---
title: 'Java e Bedrock no mesmo mundo: um servidor de Minecraft na AWS'
description: 'Meu irmão joga no tablet, eu jogo no PC, e as duas versões do Minecraft não conversam. O que era uma tarde de jogo virou um servidor em EC2, um tradutor de protocolo e umas boas horas de depuração.'
pubDate: 2026-09-13
tags: ['linux', 'aws', 'seguranca', 'java']
draft: false
---

*EC2 t3.medium · Amazon Linux 2023 · Geyser standalone · 2 jogadores*

## O problema não era a porta

Comecei do jeito óbvio: abri meu mundo single player, cliquei em "Abrir para LAN", peguei a porta e mandei pro tablet do meu irmão. Ele estava na mesma rede. Não conectou.

Passei um tempo mexendo em firewall e porta antes de entender o que realmente estava acontecendo: **Java Edition e Bedrock Edition não são o mesmo jogo em redes diferentes — são protocolos diferentes.** O Java fala TCP com um protocolo próprio; o Bedrock fala UDP sobre RakNet. O servidor que o "Abrir para LAN" levanta só entende o protocolo Java. O cliente Bedrock não tem como sequer descobrir aquele servidor, muito menos negociar conexão. Nenhuma configuração de rede resolve isso, porque não é problema de rede.

> **Lição:** quando nenhuma variação da sua hipótese funciona, provavelmente a hipótese está errada. Eu estava depurando rede num problema de protocolo.

## Geyser: um tradutor no meio

A solução do ecossistema é o [Geyser](https://geysermc.org): um proxy que escuta conexões Bedrock e as traduz para o protocolo Java, conectando-se ao servidor como se fosse um cliente Java comum. Ele não sabe nada sobre o jogo em si — não guarda mundo, não valida nada. Só traduz pacotes nas duas direções.

Ele existe em várias formas: plugin de Spigot/Paper, mod de Fabric, extensão de proxies como BungeeCord e Velocity, e uma versão **standalone**, que roda como processo independente. Escolhi a standalone porque eu queria manter o servidor vanilla, sem plugins.

A parte que importa entender é a autenticação, e é onde eu tropecei depois. O Geyser tem três modos no `auth-type`: `online` exige que o jogador Bedrock vincule uma conta Java de verdade; `floodgate` usa um plugin companheiro para identificar o jogador pela conta Bedrock dele; e `offline` simplesmente não verifica nada — o jogador entra com o gamertag como nome. Para dois irmãos jogando um mundo privado, `offline` é o modo pensado exatamente para isso.

## Tentativa 1 (local): o "Abrir para LAN" não serve de backend

Configurei o Geyser no meu PC apontando para a porta que o mundo aberto em LAN expunha. O tablet chegou até o Geyser, e o Geyser levou um tapa na cara do servidor:

```text
Cannot reply to ClientboundHelloPacket without profile and access token
```

O motivo: o mundo aberto em LAN roda com `online-mode=true`, herdado da minha sessão autenticada. Ou seja, ele exige que qualquer cliente que conecte prove ter uma conta Java válida contra os servidores da Mojang. O Geyser não tem conta Java nenhuma para apresentar — ele está representando um jogador de Bedrock. E o menu do jogo não oferece nenhuma forma de desligar esse modo.

Daí a primeira mudança estrutural: trocar o mundo single player por um **servidor dedicado de verdade**, onde eu controlo o `server.properties`.

```bash
# servidor dedicado, o essencial
java -jar server.jar nogui        # 1ª execução: gera arquivos e para
echo "eula=true" > eula.txt       # aceita a licença

# em server.properties
online-mode=false                 # deixa o Geyser entrar
level-name=world                  # pasta do mundo carregado
```

Copiei meu save de `%appdata%\.minecraft\saves` para a pasta do servidor como `world` e subi. Funcionou — os dois entraram. Mas agora eu tinha um servidor amarrado à minha máquina ligada, num IP residencial dinâmico, atrás de um firewall doméstico. Hora de tirar isso da minha mesa.

> **Lição:** o `auth-type` do Geyser e o `online-mode` do servidor são travas independentes. Uma cuida do lado Bedrock, a outra do lado Java, e as duas precisam estar coerentes.

## Tentativa 2 (nuvem): a arquitetura na EC2

Subi uma `t3.medium` (2 vCPU, 4 GB) com Amazon Linux 2023. Para dois jogadores é confortável: o servidor Java fica com 3 GB de heap e o Geyser usa cerca de 500 MB. Uma `t3.small` roda, mas apertada.

<figure>
  <svg viewBox="0 0 800 210" role="img" aria-label="Diagrama: cliente Java conecta na porta TCP 25565 do servidor; cliente Bedrock conecta na porta UDP 19132 do Geyser, que repassa para 127.0.0.1 na porta 25565." xmlns="http://www.w3.org/2000/svg">
    <defs>
      <marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto">
        <path d="M0,0 L10,5 L0,10 z" fill="var(--color-accent)"></path>
      </marker>
    </defs>
    <rect x="608" y="14" width="180" height="182" fill="var(--color-bg)" stroke="var(--color-border)" stroke-width="1.5"></rect>
    <text x="698" y="38" text-anchor="middle" font-family="var(--font-mono)" font-size="10.5" font-weight="600" letter-spacing="1.4" fill="var(--color-text-muted)">INSTÂNCIA EC2</text>
    <rect x="10" y="28" width="168" height="48" fill="var(--color-surface)" stroke="var(--color-border)" stroke-width="1.5"></rect>
    <text x="94" y="49" text-anchor="middle" font-family="var(--font-sans)" font-size="12.5" font-weight="600" fill="var(--color-text)">Cliente Java</text>
    <text x="94" y="66" text-anchor="middle" font-family="var(--font-mono)" font-size="10.5" fill="var(--color-text-muted)">PC</text>
    <rect x="10" y="134" width="168" height="48" fill="var(--color-surface)" stroke="var(--color-border)" stroke-width="1.5"></rect>
    <text x="94" y="155" text-anchor="middle" font-family="var(--font-sans)" font-size="12.5" font-weight="600" fill="var(--color-text)">Cliente Bedrock</text>
    <text x="94" y="172" text-anchor="middle" font-family="var(--font-mono)" font-size="10.5" fill="var(--color-text-muted)">tablet</text>
    <rect x="378" y="134" width="142" height="48" fill="var(--color-surface)" stroke="var(--color-accent)" stroke-width="1.5"></rect>
    <text x="449" y="155" text-anchor="middle" font-family="var(--font-sans)" font-size="12.5" font-weight="600" fill="var(--color-text)">Geyser</text>
    <text x="449" y="172" text-anchor="middle" font-family="var(--font-mono)" font-size="10.5" fill="var(--color-text-muted)">tradutor</text>
    <rect x="632" y="80" width="132" height="56" fill="var(--color-surface)" stroke="var(--color-accent)" stroke-width="1.5"></rect>
    <text x="698" y="103" text-anchor="middle" font-family="var(--font-sans)" font-size="12.5" font-weight="600" fill="var(--color-text)">server.jar</text>
    <text x="698" y="120" text-anchor="middle" font-family="var(--font-mono)" font-size="10.5" fill="var(--color-text-muted)">mundo</text>
    <g stroke="var(--color-accent)" stroke-width="1.5" fill="none">
      <path d="M178,52 L626,52 L626,84" marker-end="url(#arrow)"></path>
      <path d="M178,158 L372,158" marker-end="url(#arrow)"></path>
      <path d="M520,158 L586,158 L586,122 L626,122" marker-end="url(#arrow)"></path>
    </g>
    <text x="400" y="42" text-anchor="middle" font-family="var(--font-mono)" font-size="10.5" fill="var(--color-accent)">TCP 25565</text>
    <text x="275" y="149" text-anchor="middle" font-family="var(--font-mono)" font-size="10.5" fill="var(--color-accent)">UDP 19132</text>
    <text x="562" y="112" text-anchor="middle" font-family="var(--font-mono)" font-size="10" fill="var(--color-text-muted)">127.0.0.1</text>
  </svg>
  <figcaption style="font-size:0.85rem;color:var(--color-text-muted);margin-top:0.6rem;">Os dois processos vivem na mesma instância, então o Geyser fala com o servidor por <code>127.0.0.1</code> — não precisa do IP público em nenhuma configuração.</figcaption>
</figure>

No security group, duas regras de entrada além do SSH: **TCP 25565** para o Java e **UDP 19132** para o Bedrock. Esse detalhe engana: o console da AWS oferece "Custom TCP" primeiro, e é fácil criar a regra do Bedrock no protocolo errado. Bedrock é UDP, sempre.

O SSH eu deixei restrito ao meu IP. E vale saber de antemão: sem Elastic IP, o endereço público muda a cada ciclo de stop/start — o que significa reconfigurar os dois clientes. Um `reboot` preserva.

### Transferir o mundo

O mundo é só uma pasta. Compactar, enviar, extrair:

```powershell
# no Windows (PowerShell)
Compress-Archive -Path .\world -DestinationPath world.zip
scp -i chave.pem world.zip ec2-user@<ip>:~/

# na instância
sudo unzip ~/world.zip -d /opt/minecraft/server/
sudo chown -R mcuser:mcuser /opt/minecraft/server/world
```

Aquele `chown` no fim não é detalhe: extrair com `sudo` cria os arquivos como root, e o serviço roda como usuário sem privilégio. Sem isso o servidor morre com `AccessDeniedException` antes de carregar o primeiro chunk.

### Duas JVMs, não uma

O `server.jar` atual exige Java 25. O Geyser roda em Java 21. Em vez de brigar com isso, instalei os dois Corretto e cada serviço aponta para o seu binário:

```bash
sudo dnf install -y java-21-amazon-corretto java-25-amazon-corretto \
    jq unzip curl --allowerasing
```

O `--allowerasing` resolve um conflito bem específico do Amazon Linux 2023: a distro já vem com `curl-minimal`, que colide com o pacote `curl` completo. Sem a flag, o `dnf` aborta a transação inteira.

### systemd como supervisor

Rodar `java -jar` numa sessão SSH funciona até você fechar o terminal. Com systemd, cada processo vira serviço: sobe no boot, religa se cair, e fica independente de sessão.

```ini
# /etc/systemd/system/mc-server.service
[Service]
User=mcuser
WorkingDirectory=/opt/minecraft/server
ExecStart=/usr/lib/jvm/java-25-amazon-corretto.x86_64/bin/java \
    -Xmx3G -Xms2G -jar server.jar nogui
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

O preço é perder o console interativo: não existe onde digitar `/whitelist` ou `/ban`. A saída é gerenciar por arquivo, ou — muito melhor — se declarar operador no `ops.json` e usar os comandos de dentro do jogo.

> **Lição:** serviço sem console não é serviço sem administração. Ser operador nível 4 no próprio jogo substitui o acesso por SSH para quase tudo do dia a dia.

## Quatro erros que mentiram para mim

Nenhum dos problemas abaixo foi difícil de corrigir. Todos foram difíceis de *encontrar*, porque a mensagem de erro apontava para o lugar errado.

### `Network is unreachable: getsockopt`

Esse me custou mais tempo que todo o resto somado. Testei IPv6, Teredo, catálogo Winsock, VPN, reproduzi em outra máquina. O log do cliente entregou: `Connecting to 25565, 25565`. O campo de endereço tinha só a porta, sem IP — e o Java interpreta número puro como IPv4 em notação inteira, então `25565` virou `0.0.99.221`, endereço sem rota. O erro era de digitação.

### `UnsupportedClassVersionError: class file version 69.0`

Versão de bytecode incompatível. A conta é simples: subtraia 44 e você tem a versão do Java. 65 é Java 21, 69 é Java 25. Foi assim que descobri que precisava das duas JVMs.

### `status=203/EXEC`

O systemd não conseguiu nem executar o binário. O Corretto 21 instala em `java-21-amazon-corretto.x86_64` e, diferente do 25, não cria symlink sem o sufixo — o caminho que eu tinha escrito no `ExecStart` simplesmente não existia.

### Crash-loop com erro de parse YAML

Um `sed -i` meu para trocar `auth-type: online` por `offline` removeu a indentação da linha. Como a chave vive dentro do bloco `java:`, tirar os dois espaços encerrou o mapeamento e deixou a chave seguinte órfã. Em YAML, indentação é sintaxe.

> **Lição:** antes de qualquer `sed -i`, `cp -a arquivo arquivo.bak`. E confira o resultado com `sed -n` ou `grep` — nunca presuma que a substituição saiu como você imaginou.

## Alguém entrou no meu servidor

Poucas horas depois de subir, apareceu um nick desconhecido no log, vindo de um IP de hosting europeu. Não foi sorte dele: existem scanners varrendo a internet em busca de servidores Minecraft na porta 25565 continuamente. Com a porta aberta para `0.0.0.0/0` e `online-mode=false`, meu servidor era efetivamente público e sem autenticação — qualquer pessoa escolhia o nick que quisesse e entrava.

A correção certa é a whitelist. E aqui tem uma pegadinha do offline mode: sem a Mojang dizendo qual é o UUID de cada jogador, o servidor deriva um localmente, do MD5 de `OfflinePlayer:<nome>`. Então a `whitelist.json` precisa do UUID calculado exatamente assim. Dá para gerar com o próprio Java:

```java
java.util.UUID.nameUUIDFromBytes(
    ("OfflinePlayer:" + nick).getBytes(StandardCharsets.UTF_8))
```

Com o arquivo no lugar, duas chaves no `server.properties` fecham a porta: `white-list=true` ativa a lista, e `enforce-whitelist=true` expulsa na hora quem já estiver conectado e não estiver nela.

### O furo que eu não tinha visto

Revisando o setup, achei algo pior que o intruso. Os dois serviços rodavam como `ec2-user` — que no Amazon Linux tem **sudo sem senha**. Ou seja, qualquer falha de parsing no Geyser ou no servidor, que processam protocolo binário de qualquer pessoa da internet sem autenticação prévia, não daria um usuário limitado ao atacante: daria root, a um `sudo` de distância.

```bash
# usuário dedicado + sandbox
sudo useradd -r -s /usr/sbin/nologin -d /opt/minecraft mcuser
sudo chown -R mcuser:mcuser /opt/minecraft

# no [Service] dos dois units
User=mcuser
NoNewPrivileges=true
ProtectSystem=strict
ReadWritePaths=/opt/minecraft
```

Agora execução de código nesses processos cai num usuário sem shell e sem sudo, que só escreve em `/opt/minecraft`. O `ProtectSystem=strict` deixa o resto do disco somente leitura para o serviço, e o `NoNewPrivileges` impede escalada via binários setuid.

Completei com um budget alarm na AWS com alerta em valor baixo. O risco financeiro real de uma instância comprometida não é o mundo do Minecraft — é a fatura de mineração de cripto.

> **Lição:** whitelist bloqueia quem entra pela porta da frente; ela não protege o parser, que é exercitado antes de qualquer login. Reduzir o privilégio do processo é a camada que continua valendo quando a primeira falha.

## Os comandos que eu mais usei

Praticamente toda a operação do dia a dia se resume a meia dúzia:

| Comando | O que faz |
| --- | --- |
| `systemctl status mc-server` | Estado, PID, memória e as últimas linhas de log |
| `sudo systemctl restart mc-server` | O que você roda depois de mexer em qualquer config |
| `sudo systemctl daemon-reload` | Obrigatório após editar um `.service` — sem isso o systemd usa a versão antiga |
| `sudo systemctl enable mc-server` | Sobe no boot. Não confundir com `start`: são eixos diferentes |
| `journalctl -u mc-server -f` | Log ao vivo; `Ctrl+C` sai do acompanhamento sem derrubar o serviço |
| `journalctl -u mc-server -n 50 --no-pager` | Últimas linhas direto na tela, sem abrir o pager |
| `grep -n "auth-type" config.yml` | Localiza a chave e o número da linha antes de editar |
| `sed -n '24,40p' config.yml` | Imprime um intervalo — o jeito de conferir o que o `sed -i` fez |
| `sudo chown -R mcuser:mcuser /opt/minecraft` | Depois de qualquer extração ou cópia feita como root |
| `tar -czf world.tar.gz -C /opt/minecraft/server world` | Backup do mundo; o `-C` evita caminhos absolutos dentro do pacote |

## Quanto custa

A conta muda completamente dependendo de você deixar a instância ligada ou não. Para duas pessoas jogando algumas horas por semana, desligar entre as sessões é o que torna isso barato:

| Item | Observação |
| --- | --- |
| t3.medium | Cobrada por hora de execução. Parada, não custa nada |
| EBS 8 GiB | Continua sendo cobrado com a instância parada — centavos por mês |
| IPv4 público | Cobrado por hora desde 2024, seja dinâmico ou Elastic IP |
| Elastic IP | Vale a pena se você liga/desliga muito: evita reconfigurar os clientes |

Como os serviços estão `enabled`, ligar a instância é suficiente: os dois sobem sozinhos e o mundo carrega. Não preciso nem abrir SSH para jogar.

## O que eu faria diferente

1. Subir direto em `sa-east-1`. Comecei em Ohio por inércia e paguei em latência — para jogadores no Brasil, São Paulo é obviamente melhor.
2. Ativar a whitelist no primeiro boot, antes de a porta ficar aberta. Os scanners chegam em horas, não em dias.
3. Criar o usuário dedicado desde o começo, em vez de deixar o serviço rodando como `ec2-user` e corrigir depois.
4. Considerar Paper com o Geyser como plugin em vez do standalone. Um processo só, e abre a porta para Multiverse se eu quiser vários mundos simultâneos.
5. Elastic IP desde o início. O custo é trivial perto do incômodo de avisar o IP novo toda vez.

No fim, o servidor que era para resolver uma tarde de jogo virou o melhor laboratório de Linux e AWS que eu tive em muito tempo. Errar em produção de verdade ensina; errar num servidor de Minecraft ensina quase o mesmo e não acorda ninguém de madrugada.

---

*Stack: EC2 t3.medium · Amazon Linux 2023 · Corretto 25 e 21 · Minecraft Java Edition vanilla · Geyser standalone · systemd.*

---
title: 'Garbage Collectors da JVM: G1, ZGC e quando cada um faz sentido'
description: 'Um tour pelos coletores de lixo modernos da JVM — como funcionam, qual o trade-off entre throughput e latência, e como escolher o certo para sua carga.'
pubDate: 2024-02-01
tags: ['java', 'jvm', 'performance']
draft: false
---

A maior parte dos desenvolvedores Java trata o **garbage collector** como uma
caixa-preta — até a primeira pausa de GC de 2 segundos em produção. Entender o
que acontece por baixo é o que separa "aumentei o heap e rezei" de uma decisão
de tuning consciente. Vamos abrir a caixa.

## O problema fundamental

Todo GC precisa equilibrar três coisas que estão em tensão:

- **Throughput** — quanto do tempo de CPU é trabalho útil vs. coleta.
- **Latência** — o tamanho das pausas (stop-the-world) que a aplicação sofre.
- **Footprint** — quanta memória e CPU extra o coletor consome.

Você não otimiza os três ao mesmo tempo. Escolher um GC é escolher qual eixo
sacrificar.

## G1 — o padrão pragmático

Desde o Java 9, o **G1 (Garbage-First)** é o coletor padrão. Ele divide o heap
em regiões de tamanho fixo e coleta primeiro as que têm mais lixo — daí o nome.
O objetivo dele é cumprir uma meta de pausa que você define:

```java
// Mira pausas de no máximo ~50ms; o G1 ajusta o resto sozinho.
// -XX:+UseG1GC -XX:MaxGCPauseMillis=50
```

G1 é a escolha segura para a maioria das aplicações de servidor: bom
throughput, pausas previsíveis na casa das dezenas de milissegundos.

## ZGC — quando a latência é sagrada

O **ZGC** foi desenhado para um único objetivo: pausas **sub-milissegundo**,
mesmo em heaps de centenas de gigabytes. Ele faz quase todo o trabalho
concorrentemente com a aplicação, usando *colored pointers* e *load barriers*
para mover objetos sem parar o mundo.

```java
// Pausas que não crescem com o tamanho do heap.
// -XX:+UseZGC
```

O trade-off: ZGC consome mais CPU e memória que o G1. Você paga em throughput
e footprint para comprar latência. Faz sentido para sistemas onde uma pausa de
100ms é inaceitável — trading de baixa latência, APIs com SLA agressivo de p99.

## Como escolher

Um guia rápido, sem dogma:

| Carga | Coletor |
| --- | --- |
| App de servidor típico | **G1** (padrão) |
| Latência p99 crítica, heap grande | **ZGC** |
| Batch / throughput puro | **Parallel GC** |

## Meça antes de tunar

A regra de ouro: **não troque de GC no escuro**. Ligue os logs e olhe o
comportamento real antes de mexer em qualquer flag:

```bash
# Logs unificados de GC a partir do Java 9+
java -Xlog:gc*:file=gc.log:time,uptime -jar app.jar
```

Ferramentas como o [GCeasy](https://gceasy.io) transformam esse log em
gráficos de pausa e throughput. Na imensa maioria dos casos, o G1 com um
`MaxGCPauseMillis` razoável já resolve — e você descobre isso medindo, não
adivinhando.

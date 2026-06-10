---
title: 'SOLID na prática: além das definições de manual'
description: 'Os cinco princípios SOLID explicados com exemplos reais em Java — quando aplicar, quando relaxar e o que cada um realmente protege.'
pubDate: 2024-01-15
tags: ['java', 'arquitetura', 'solid']
draft: false
---

Quase todo mundo decora a sigla **SOLID** para entrevistas, mas poucos param
para pensar no que cada letra está realmente protegendo. SOLID não é sobre
seguir regras — é sobre controlar o custo da mudança. Vamos passar pelos cinco
princípios com código Java, focando na intuição por trás de cada um.

## S — Single Responsibility Principle

Uma classe deve ter **um, e apenas um, motivo para mudar**. O erro clássico é
misturar regra de negócio com detalhe de infraestrutura:

```java
// ❌ Faz cálculo E persistência E formatação de e-mail
public class Invoice {
    public BigDecimal total() { /* regra de negócio */ }
    public void saveToDatabase() { /* detalhe de infra */ }
    public String toEmailBody() { /* detalhe de apresentação */ }
}
```

Cada responsabilidade muda por um motivo diferente — e por uma pessoa diferente
do time. Separe-as:

```java
public class Invoice {
    public BigDecimal total() { /* só regra de negócio */ }
}

public class InvoiceRepository { /* persistência */ }
public class InvoiceEmailFormatter { /* apresentação */ }
```

## O — Open/Closed Principle

Entidades devem estar **abertas para extensão, fechadas para modificação**. Na
prática: você adiciona comportamento novo sem editar o código existente.

```java
public interface DiscountPolicy {
    BigDecimal apply(BigDecimal amount);
}

// Um novo tipo de desconto é uma classe nova — não um if a mais.
public class BlackFridayDiscount implements DiscountPolicy {
    public BigDecimal apply(BigDecimal amount) {
        return amount.multiply(new BigDecimal("0.70"));
    }
}
```

## L — Liskov Substitution Principle

Um subtipo precisa ser usável **em qualquer lugar onde o tipo base é esperado**,
sem surpresas. Se uma subclasse lança exceção onde a base funcionava, você
quebrou Liskov. O exemplo canônico é o `Square extends Rectangle` — mexer na
largura do quadrado muda a altura, violando a expectativa de quem usa
`Rectangle`.

## I — Interface Segregation Principle

Melhor **várias interfaces específicas** do que uma genérica e gorda. Ninguém
deve ser forçado a implementar métodos que não usa:

```java
// ❌ Quem só lê é obrigado a implementar write()
interface Repository<T> {
    T findById(long id);
    void save(T entity);
    void delete(long id);
}

// ✅ Segregado por capacidade
interface ReadRepository<T> { T findById(long id); }
interface WriteRepository<T> { void save(T entity); }
```

## D — Dependency Inversion Principle

Módulos de alto nível **não devem depender de detalhes** — ambos dependem de
abstrações. É o que torna seu núcleo de negócio testável e independente do
banco, do broker ou do framework.

```java
public class CheckoutService {
    private final PaymentGateway gateway; // interface, não Stripe direto

    public CheckoutService(PaymentGateway gateway) {
        this.gateway = gateway;
    }
}
```

## O ponto que fica

SOLID é um conjunto de **heurísticas**, não dogma. O objetivo final é sempre o
mesmo: deixar o código barato de mudar quando o mundo muda — e o mundo sempre
muda. Aplique com julgamento, não no piloto automático.

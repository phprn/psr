---
title: PRS-2
category: PSRs
order: 3
---

## 1. Resumo

A intenção deste guia é reduzir o atrito cognitivo ao escanear códigos de diferentes autores. Para isso, são enumerados um conjunto compartilhado de regras e expectativas sobre como formatar o código PHP

As regras de estilo aqui são derivadas de semelhanças entre os vários projetos de membros. Quando vários autores colaboram em vários projetos, ajuda ter um conjunto de diretrizes a serem usadas entre todos esses projetos. Assim, o benefício deste guia não está nas próprias regras, mas no compartilhamento dessas regras.

## 2. Votos

- **Votação de Aceitação:** [ML](https://groups.google.com/d/msg/php-fig/c-QVvnZdMQ0/TdDMdzKFpdIJ)

## 3. Errata

### 3.1 - Argumentos de Várias Linhas (08/09/2013)

O uso de um ou mais argumentos de várias linhas (ou seja, arrays ou funções anônimas) não constitui 
a divisão da própria lista de argumentos, portanto, a Seção 4.6 não é aplicada automaticamente.
Matrizes e funções anônimas são capazes de abranger várias linhas.

Os exemplos a seguir são perfeitamente válidos no PSR-2:

~~~php
<?php
somefunction($foo, $bar, [
    // ...
], $baz);

$app->get('/hello/{name}', function ($name) use ($app) {
    return 'Hello '.$app->escape($name);
});
~~~

### 3.2 - Estendendo Múltiplas Interfaces (17/10/2013)
Ao estender várias interfaces, a lista de `extends` deve ser tratada da mesma forma que uma lista de `implements`, 
conforme declarado na Seção 4.1.

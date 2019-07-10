---
title: PSR-1 Padrão Básico de Codificação
category: PSRs
order: 1
---

# Normas Básicas de Codificação

Nesta seção da norma, compreende-se o que deve ser considerado elementos básicos de codificação
que são necessários para garantir um alto nível de interoperabilidade técnica entre códigos PHP
compartilhados

As palavras-chave "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" nesse documento 
devem ser interpretadas como descrito na [RFC 2119].

#### Tradução das Palavras-chave
"MUST" (DEVE);
"MUST NOT" (NÃO DEVE);
"REQUIRED" (OBRIGATÓRIO);
"SHALL" (TEM QUE);
"SHALL NOT" (NÃO TEM QUE);
"SHOULD" (DEVERIA);
"SHOULD NOT" (NÃO DEVERIA);
"RECOMMENDED" (RECOMENDADO);
"MAY" (PODE);
"OPTIONAL" (OPCIONAL).

[RFC 2119]: http://www.ietf.org/rfc/rfc2119.txt
[PSR-0]: https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-0.md
[PSR-4]: https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-4-autoloader.md
[BOM]: https://www.w3.org/International/questions/qa-byte-order-mark

## 1. Visão Geral

- Arquivos DEVEM usar apenas `<?php` and `<?=` tags.

- Arquivos DEVEM usar apenas UTF-8 sem [BOM] para código PHP.

- Arquivos DEVERIAM *ou* declarar símbolos (classes, funções, constantes, etc.)
  *ou* causar efeitos colaterais (por exemplo: gerar saída, modificar configurações .ini e etc.)
  mas NÃO DEVERIAM fazer os dois.

- Namespaces e classes DEVEM seguir um "autoloading" PSR: [[PSR-0], [PSR-4]].

- Os nomes das classes DEVEM ser declarados em `StudlyCaps`.

- Constantes de classe DEVEM ser declaradas em caixa alta separadas por underscore

- Os nomes dos métodos DEVEM ser declarados em `camelCase`.

## 2. Arquivos

### 2.1. PHP Tags

Código PHP DEVE usar a tag `<?php ?>` longa ou a tag echo-curto `<?= ?>`;
NÃO DEVE ser utilizada outras variações de tag.

### 2.2. Codificação de caracteres

Código PHP DEVE utilizar apenas UTF-8 sem [BOM].

### 2.3. Efeitos colaterais

Um arquivo DEVERIA declarar novos símbolos (classes, funções, constantes, etc) e não causar outros efeitos colaterais, 
ou DEVERIA executar lógica com efeitos colaterais, mas NÃO DEVERIA fazer as duas coisas.

A frase "efeito colateral" significa a execução de lógica não diretamente relacionada a declarar classes, funções, constantes, etc.,
*simplesmente a partir do arquivo incluído*

"Efeito colateral" não se limita apenas a: gerar saídas, uso explícito de `require` ou `include`, conectando-se serviços externos, modificando configurações ini, emitindo erros ou exceções, modificando variáveis globais ou estáticas,
lendo ou escrevendo em arquivos e assim por diante.

O exemplo a seguir contém efeitos colaterais e declarações; ou seja, um exemplo a se evitar

~~~php
<?php
// efeito colateral: modificando configuração ini
ini_set('error_reporting', E_ALL);

// efeito colateral: carregando um arquivo
include "file.php";

// efeito colateral: gerando saída
echo "<html>\n";

// declaração
function foo()
{
    // corpo da função
}
~~~


A seguir, um exemplo de um arquivo contendo instruções sem efeitos colaterais; ou seja, um exemplo a seguir:

~~~php
<?php
// declaração
function foo()
{
    // corpo da função
}

// declaração condicional *não* é um efeito colateral
if (! function_exists('bar')) {
    function bar()
    {
        // corpo da função
    }
}
~~~

## 3. Namespace e Nomes de Classes

Namespaces e classes DEVEM seguir um "autoloading" PSR: [[PSR-0], [PSR-4]].

Isso significa que cada classe está em em um arquivo por si só
e está em um namespace de um nível a menos: nível superior do vendor

Nomes de classes DEVEM ser declarados em `StudlyCaps`.

Código escrito em PHP 5.3 e posterior DEVEM usar namespaces formais.

Por exemplo:

~~~php
<?php
// PHP 5.3 e porterior:
namespace Vendor\Model;

class Foo
{
}
~~~

Código escrito em PHP 5.2.x e anterior DEVERIAM usar a convenção de pseudo-nampesace
nos prefixos `Vendor_` no nome da classe.

~~~php
<?php
// PHP 5.2.x e anterior:
class Vendor_Model_Foo
{
}
~~~

## 4. Constantes de Classe, Propriedades e Métodos

O termo "classe" se refere a todas as classes, interfaces e traits.

### 4.1. Constantes

Constantes de classe DEVEM ser declararas todas em caixa alta separadas por underscore.
Por exemplo:

~~~php
<?php
namespace Vendor\Model;

class Foo
{
    const VERSION = '1.0';
    const DATE_APPROVED = '2012-06-01';
}
~~~

### 4.2. Propriedades

Este guia intencionalmente evita qualquer recomendação sobre o uso de
`$StudlyCaps`, `$camelCase`, or `$under_score` para nomes de propriedades.


Seja qual for a convenção de nomenclatura usada, DEVERIA ser aplicada consistentemente dentro de um
escopo razoável. Esse escopo pode ser em nível-de-vendor, nível-de-pacote, nível-de-classe,
ou nível de método.

### 4.3. Métodos

Nomes de métodos DEVEM ser declarados em `camelCase()`.

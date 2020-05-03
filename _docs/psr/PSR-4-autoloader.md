---
title: PSR-4 Autoloader
category: PSRs
order: 5
---

# Autoloader

As palavras-chave "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" nesse documento devem ser
interpretadas como descrito na [RFC 2119](http://tools.ietf.org/html/rfc2119).

## 1. Visão geral

Esse PSR descreve a especificação para a classe [autoloading][].
É totalmente interoperável. Esse PSR também descreve onde colocar
os arquivos que serão carregados automaticamente de acordo com
a espeficação.

## 2. Especificação

1. O termo "classe" refere-se a classes, interfaces, traits, and 
   outras estruturas similares.

2. Um nome de classe totalmente qualificado posssui o seguinte formulário:

        \<NamespaceName>(\<SubNamespaceNames>)*\<ClassName>

    1. Um nome de classe totalmente qualificado DEVE ter um namespace de nível superior,
       também conhecido como "vendor namespace".

    2. Um nome de classe totalmente qualificado PODE ter um ou mais nomes de sub-namespaces.

    3. Um nome de classe totalmente qualificado DEVE ter uma terminação para o nome da classe.

    4. Underscores não tem significado algum em qualquer porção do nome de classe 
       totalmente qualificado.

    5. Caracteres alfabéticos em um nome de classe totalmente qualificado PODEM ser 
       qualquer combinação de letras maísculas e minúsculas.

    6. Todos os nomes de classes DEVEM ser referenciados de modo case-sensitive.

3. Ao carregar um arquivo que corresponde a um nome de classe totalmente qualificado ...

    1. Uma série contígua de um ou mais nomes principais de namespaces e sub-namespaces, 
       sem incluir o separador de namespace inicial, em um nome de classe totalmente 
       qualificado (um "prefixo de namespace"), corresponde a pelo menos um
       "diretório base".

    2. Os nomes de sub-namespaces contíguos após o "prefixo de namespace" correspondem
       a um subdiretório dentro de um "diretório base", no qual os separadores de 
       namespace representam separadores de diretório. O nome do subdiretório DEVE
       corresponder ao caso dos nome do sub-namespace.

    3. A terminação no nome de uma classe corresponde a um nome de arquivo que termina com `.php`.
       O nome do arquivo DEVE corresponder ao caso da terminação no nome de uma classe.

4. Implementações de Autoloader NÃO DEVEM lançar exceções, NÃO DEVEM mostrar erros
   de qualquer nível e NÃO DEVEM retornar um valor.

## 3. Exemplos

A tabela abaixo mostra o caminho correspondente para um determinado nome de classe 
totalmente qualificado, prefixo de namespace and diretório base.

| Nome de Classe Totalmente Qualificado | Prefixo de Namespace | Diretório Base           | Resulting File Path                       |
| --------------------------------------|----------------------|--------------------------|-------------------------------------------|
| \Acme\Log\Writer\File_Writer          | Acme\Log\Writer      | ./acme-log-writer/lib/   | ./acme-log-writer/lib/File_Writer.php     |
| \Aura\Web\Response\Status             | Aura\Web             | /path/to/aura-web/src/   | /path/to/aura-web/src/Response/Status.php |
| \Symfony\Core\Request                 | Symfony\Core         | ./vendor/Symfony/Core/   | ./vendor/Symfony/Core/Request.php         |
| \Zend\Acl                             | Zend                 | /usr/includes/Zend/      | /usr/includes/Zend/Acl.php                |

Para exemplos de implementação de autoloaders conforme a especificação, por favor, acesse [examples file][].
Exemplos de implementações NÃO DEVEM ser considerados como parte da especificação e PODEM mudar a qualquer momento.

[autoloading]: http://php.net/autoload
[examples file]: https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-4-autoloader-examples.md

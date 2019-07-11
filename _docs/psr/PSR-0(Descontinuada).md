---
title: PSR-0 Descontinuada
category: PSRs
order: 0
---

Padrão de Carregamento Automático (AutoLoading Standard)
====================

> **Obsoleto** - A partir de 21-10-2014, o PSR-0 foi marcado como obsoleto. Agora o [PSR-4] é recomendado
como uma alternativa.

[PSR-4]: http://www.php-fig.org/psr/psr-4/
[totalmente-qualificados]: https://www.php.net/manual/pt_BR/language.namespaces.rules.php

Abaixo estão os requisitos obrigatórios que devem ser cumpridos para interoperabilidade autoloader.

Obrigatório
-----------

* Um namespace e classe [totalmente-qualificados] devem ter as seguintes
  estrutura `\<Vendor Name>\(<Namespace>\)*<Class Name>`
* Cada namespace deve ter um namespace de nível superior ("Vendor Name").
* Cada namespace pode ter tantos sub-namespaces quanto desejar.
* Cada separador de namespace é convertido em um `SEPARADOR_DE_DIRETORIO` ao
  carregar do sistema de arquivos.
* Cada caractere `_` no NOME DA CLASSE é convertido em um `SEPARADOR_DE_DIRETORIO`. 
  O caractere `_` não tem significado especial no namespace.
* O namespace e a classe totalmente qualificados são sufixados com `.php` ao
  carregar do sistema de arquivos.
* Caracteres alfabéticos em vendor names, namespaces e nomes de classes podem
  ter qualquer combinação de minúsculas e maiúsculas.

Exemplos
--------

* `\Doctrine\Common\IsolatedClassLoader` => `/path/to/project/lib/vendor/Doctrine/Common/IsolatedClassLoader.php`
* `\Symfony\Core\Request` => `/path/to/project/lib/vendor/Symfony/Core/Request.php`
* `\Zend\Acl` => `/path/to/project/lib/vendor/Zend/Acl.php`
* `\Zend\Mail\Message` => `/path/to/project/lib/vendor/Zend/Mail/Message.php`

Underlines em Namespaces e Nomes de Classes
-------------------------------------------

* `\namespace\package\Class_Name` => `/path/to/project/lib/vendor/namespace/package/Class/Name.php`
* `\namespace\package_name\Class_Name` => `/path/to/project/lib/vendor/namespace/package_name/Class/Name.php`

Os padrões que definimos aqui devem ser o menor denominador comum para
interoperabilidade indolor do autoloader. Você pode testar se está
seguindo esses padrões utilizando o exemplo de implementação abaixo da 
SplClassLoader que é capaz de carregar classes PHP 5.3

Exemplo de Implementação
------------------------

Abaixo está uma função para demonstrar o autoloader dos padrões propostos acima.

~~~php
<?php

function autoload($className)
{
    $className = ltrim($className, '\\');
    $fileName  = '';
    $namespace = '';
    if ($lastNsPos = strrpos($className, '\\')) {
        $namespace = substr($className, 0, $lastNsPos);
        $className = substr($className, $lastNsPos + 1);
        $fileName  = str_replace('\\', DIRECTORY_SEPARATOR, $namespace) . DIRECTORY_SEPARATOR;
    }
    $fileName .= str_replace('_', DIRECTORY_SEPARATOR, $className) . '.php';

    require $fileName;
}
spl_autoload_register('autoload');
~~~

Implementação da SplClassLoader
-------------------------------

O gist abaixo contém uma amostra de implementação da SplClassLoader que pode 
carregar suas classes se você seguir a interoperabilidade do autoloader dos 
padrões propostos acima. Atualmente é a maneira recomendada de carregar 
classes PHP 5.3 que seguem esses padrões.

* [http://gist.github.com/221634](http://gist.github.com/221634)

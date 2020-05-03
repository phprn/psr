---
title: PSR-3 Interface de Log
category: PSRs
order: 4
---

Interface de Log
================

Este documento descreve uma interface comum para registrar bibliotecas.

O objetivo principal é permitir que bibliotecas recebam o objeto `Psr\Log\LoggerInterface`
e gravem logs nesta, de maneira simples e universal. Frameworks e CMSs(sigla para 
Content Manage Systems - Sistemas de Gerenciamento de Conteúdo) que têm necessidades 
personalizadas PODEM estender a interface para suas próprias finalidades, mas DEVEM permanecer 
compatíveis com este documento. Isso garante que as bibliotecas de terceiros que um aplicativo 
utiliza, possam gravar nos logs de aplicativos centralizados.

As palavras-chave "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" nesse documento devem ser
interpretadas como descrito na [RFC 2119][].

A palavra `implementador` nesse documento deve ser interpretada como alguém
implementando o `LoggerInterface` em uma biblioteca ou framework relacionado
ao log. Usuários que utilizam logs são chamados de `user`.

[RFC 2119]: http://tools.ietf.org/html/rfc2119

## 1. Especificação

### 1.1 Noções básicas

- O `LoggerInterface` expõe oito método para gravar logs nos oito níveis [RFC 5424][] 
  (depuração, informações, notificação prévia, aviso, erro, crítico, alerta, emergência).

- Um nono método, `log`, aceita um nível de log como primeiro argumento. Chamar esse método 
  com uma das constantes com uma das constantes de nível de log DEVE ter o mesmo resultado 
  quando um método de nível específico é chamado. Chamando este método com um nível não
  definido por esta especificação, DEVE retornar um `Psr\Log\InvalidArgumentException` 
  se a implementação não conhecer o nível. Os usuários NÃO DEVEM usar um nível 
  personalizado sem saber ao certo se a implementação atual o suporta.

[RFC 5424]: http://tools.ietf.org/html/rfc5424

### 1.2 Messagem

- Todo Método aceita uma string como a mensagem, ou um objeto com o método
  `__toString()`. Implementadores PODEM ter tratamento especial para os 
  objectos passados. Se esse não for o caso, implementadores DEVEM lança-lo
  para uma string.

- A mensagem PODE conter placeholders que os implementadores PODEM substituir 
  por valores do array em contexto.

  Os nomes dos placeholders DEVEM corresponder as chaves do array em contexto.

  Os nomes dos placeholders DEVEM ser delimitados por uma única chave para abertura `{` e
  uma única chave de fechamento `}`. NÃO DEVE haver nenhum espaço em branco entre os
  delimitadores e o nome do placeholder.

  Os nomes dos placeholders DEVEM ser compostos apenas por caracteres `A-Z`, `a-z`,
  `0-9`, underscore `_`, e ponto `.`. O uso de outros caracteres é reservado para
  futuras modificações na especificação dos placeholders.

  Os implementores PODEM usar placeholders para implementar várias estratégias de escape
  e traduzir logs para serem exibidos. usuários NÃO DEVEM pré-escapar valores de placeholder
  pois eles não podem saber em que contexto os dados serão exibidos.

  O exemplo de implementação de interpolação de placeholder abaixo, é apenas para fins de referência:

  ~~~php
  <?php

  /**
   * Interpola os valores de contexto nos placeholders da mensagem.
   */
  function interpolate($message, array $context = array())
  {
      // Constrói um array de substituição com chaves { } em torno dos contextos chave
      $replace = array();
      foreach ($context as $key => $val) {
          // checa se o valor pode ser convertido em string
          if (!is_array($val) && (!is_object($val) || method_exists($val, '__toString'))) {
              $replace['{' . $key . '}'] = $val;
          }
      }

      // interpola os valores de substituição na messagem e retorna
      return strtr($message, $replace);
  }

  // Uma mensagem com nomes de placeholders delimitados por chaves
  $message = "User {username} created";

  // Um array de contexto com nomes de placeholders => valores de substituição
  $context = array('username' => 'bolivar');

  // mostra na tela "Usuário bolivar criado"
  echo interpolate($message, $context);
  ~~~

### 1.3 Contexto

- Todo método aceita um array como dados de contexto. Isso significa manter
  qualquer informação irrelevante que não se encaixe bem em uma string. O
  array pode conter qualquer coisa. Implementadores DEVEM garantir que os
  dados sejam tratados com o máximo de tolerância possível. Um determinado
  valor no contexto NÃO DEVE lançar exceção nem gerar erros ou avisos php.

- Se um objeto do tipo `Exception` for passado nos dados de contexto, este 
  DEVE estar na chave `'exception'`. Exceções de registro são um padrão 
  comum e isso permite que os implementadores extraiam um rastreamento da
  pilha de exceção quando o backend de logs tem suporte à isto. Implmentadores
  DEVEM ainda verificar se a chave `'exception'` é realmente uma `Exception`
  antes de usá-la como tal, pois PODE conter qualquer coisa.

### 1.4 Classes auxiliares e interfaces

- A classe `Psr\Log\AbstractLogger` permite que você implemente facilmente o
  `LoggerInterface`, estendendo-o e implementando o métido genérico `log`. Os
  Outros oito métodos estão encaminhando a mensagem e o contexto para a mesma.

- Da mesma forma, usar o `Psr\Log\LoggerTrait` requer apenas que você implmente
  o método genérico `log`. Como as traits não podem implementar interfaces,
  você ainda tem que, neste caso, implementar o `LoggerInterface`.

- O `Psr\Log\NullLogger` é fornecido junto com a interface. PODE ser utilizado
  por usuários da interface para fornecer uma implementação de "buraco negro"
  caso não seja dado nenhum registrador à eles. No entanto, o log condicional
  pode ser uma abordagem melhor se a criação de dados de contesto tiver um
  custo alto.

- O `Psr\Log\LoggerAwareInterface` contém apenas um método
  `setLogger(LoggerInterface $logger)` e pode ser utilizado por frameworks
  para conectar-se a instâncias arbitrárias com um registrador.

- O trait `Psr\Log\LoggerAwareTrait` pode ser utilizado para implementar
  uma interface equivalente facilmente em qualquer classe. Isso lhe dá
  acesso ao `$this->logger`.

- A classe `Psr\Log\LogLevel` mantém constantes para os oito níveis de log.

## 2. Pacote

As interfaces e classes descritas, bem como as classes de exceção relevantes
e um conjunto de testes para verificar as implementações são fornecidas como
parte do pacote [psr/log](https://packagist.org/packages/psr/log).

## 3. `Psr\Log\LoggerInterface`

~~~php
<?php

namespace Psr\Log;

/**

 * Descreve uma instância do registrador.
 *
 * A mensagem DEVE ser uma string ou um objeto implementado __toString().
 *
 * A mensagem PODE conter espaços placeholders no formulário: {foo} onde foo
 * será substituído pelos dados do contexto na chave "foo"
 *
 * O array de contexto pode conter dados arbitrários, a única suposição que
 * pode ser feita pelos implementadores é que, se uma instância de Exceção
 * é dada para produzir um rastreamento da pilha, este deve estar em uma 
 * chave chamada "exceção".
 *
 * Para a especificação completa da interface, acesse
 * https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-3-logger-interface.md
 */
interface LoggerInterface
{
    /**
     * O sistema está inutilizável.
     *
     * @param string $message
     * @param array $context
     * @return void
     */
    public function emergency($message, array $context = array());

    /**
     * Uma ação deve ser tomada imediatamente.
     *
     * Exemplo: Site fora do ar, banco de dados indisponível, etc.
     * Isso deve acionar os alertas de SMS e acordá-lo.
     *
     * @param string $message
     * @param array $context
     * @return void
     */
    public function alert($message, array $context = array());

    /**
     * Condições críticas.
     *
     * Exemplo: Componente de aplicativo indisponível, exceção inesperada.
     *
     * @param string $message
     * @param array $context
     * @return void
     */
    public function critical($message, array $context = array());

    /**
     * Erros de tempo de execução que não exigem ação imediata, mas
     * geralmente devem ser registrados e monitorados.
     *
     * @param string $message
     * @param array $context
     * @return void
     */
    public function error($message, array $context = array());

    /**
     * Ocorrências excepcionais que não são erros.
     *
     * Exemplo: Uso de APIs descontinuadas/obsoletas, uso inadequado de uma API, 
     * coisas indesejáveis que não são necessariamente erradas.
     *
     * @param string $message
     * @param array $context
     * @return void
     */
    public function warning($message, array $context = array());

    /**
     * Eventos normais, porém significativos.
     *
     * @param string $message
     * @param array $context
     * @return void
     */
    public function notice($message, array $context = array());

    /**
     * Eventos interessantes.
     *
     * Exemplo: Usuário efetua login, logs do SQL.
     *
     * @param string $message
     * @param array $context
     * @return void
     */
    public function info($message, array $context = array());

    /**
     * Informação detalhada de depuração.
     *
     * @param string $message
     * @param array $context
     * @return void
     */
    public function debug($message, array $context = array());

    /**
     * Logs com nível arbitrário.
     *
     * @param mixed $level
     * @param string $message
     * @param array $context
     * @return void
     */
    public function log($level, $message, array $context = array());
}
~~~

## 4. `Psr\Log\LoggerAwareInterface`

~~~php
<?php

namespace Psr\Log;

/**
 * Descreve uma instância de reconhecimento de log.
 */
interface LoggerAwareInterface
{
    /**
     * Define uma instância de log no objeto.
     *
     * @param LoggerInterface $logger
     * @return void
     */
    public function setLogger(LoggerInterface $logger);
}
~~~

## 5. `Psr\Log\LogLevel`

~~~php
<?php

namespace Psr\Log;

/**
 * Descreve os níveis de log.
 */
class LogLevel
{
    const EMERGENCY = 'emergência';
    const ALERT     = 'alerta';
    const CRITICAL  = 'crítico';
    const ERROR     = 'erro';
    const WARNING   = 'aviso';
    const NOTICE    = 'notificação prévia';
    const INFO      = 'informações';
    const DEBUG     = 'depuração';
}
~~~

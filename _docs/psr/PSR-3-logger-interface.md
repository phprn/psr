---
title: PSR-3 Interface de Log
category: PSRs
order: 4
---

Logger Interface
================

Esse documento descreve uma interface comum para registrar bibliotecas.

O objetivo principal é permitir que bibliotecas recebam um objeto
 `Psr\Log\LoggerInterface` e escrevam logs para ele de maneira simples e
 universal. Frameworks e CMSs que têm necessidades específicas PODEM estender
 a interface para seus próprios fins, mas DEVEM-SE manter compatíveis com este
 documento. Isso garante que as bibliotecas de terceiros possam escrever nos
 logs da aplicação principal.

As palavras chaves "DEVE", "NÃO DEVE", "OBRIGATÓRIO", "TEM QUE", "NÃO TEM QUE",
 "DEVERIA", "NÃO DEVERIA", "RECOMENDADO", "PODE", e "OPCIONAL" existentes nesse
 documento devem ser interpretadas como descritas na
 [RFC 2119](https://tools.ietf.org/html/rfc2119).

A palavra `implementador` deste documento é para ser interpretada como uma
 implementação de `LoggerInterface` em uma biblioteca ou framework relacionados
 a log. Usuários de loggers serão referidos como ` usuários `.

## 1. Especificação

### 1.1 Básico

- O `LoggerInterface` possui oito métodos para escrever logs em 8 níveis da
 [RFC 5424](https://tools.ietf.org/html/rfc5424)
 (debug, info, notice, warning, error, critical, alert, emergency).

- Um nono método, `log`, aceita um nível de log como primeiro argumento. Chamar
 esse método com uma das constantes DEVE ter o mesmo resultado que chamar um
 método de nível específico. Chamar esse método com um nível não definido por
 essa especificação DEVE lançar um `Psr\Log\InvalidArgumentException` se a
 implementação não conhecer o nível. Usuário NÃO DEVEM usar um nível 
 personalizado sem ter certeza se a implementação atual o suporta.

### 1.2 Mensagem

- Todo método aceita uma string como mensagem, ou um objeto com o método
 `__toString()`. Implementadores PODEM ter um manuseio especial para os objetos
 passados. Se esse não for o caso, implementadores DEVEM convertê-los para uma
 string.

- A mensagem PODE conter espaços reservados (placeholders) dos quais os
 implementadores PODEM substituir com valores de um array de contexto.

  Os nomes dos espaços reservados DEVEM corresponder com os das chaves do array
  de contexto.

  Os nomes dos espaços reservados DEVEM ser limitados com uma única chave de
   abertura `{` e uma única chave de fechamento `}`. NÃO DEVERÃO existir nenhum
   espaço em branco entre os delimitadores e os nomes dos espaços reservados.
  
  Nomes de espaços reservados DEVEM ser compostos apenas por caracteres
   `A-Z`, `a-z`, `0-9`, sublinhado `_`, e ponto `.`. O uso de outros caracteres é
   reservado para futuras modificações da especificação dos espaços reservados.
  
  Implementadores PODEM utilizar espaços reservados para implementar várias
   estratégias de saída de dados e tradução dos logs a serem exibidos. Usuários
   NÃO DEVEM fazer a saída de valores dos espaços reservados antes, visto que
   eles podem não saber em que contexto os dados serão exibidos.
  
  O que segue é um exemplo de implementação de espaços reservados fornecido
   apenas para fins de referência:

  ~~~php
  <?php

  /**
   * Realiza a interpolação dos valores de contexto nos espaços reservados da mensagem.
   */
  function interpolate($message, array $context = array())
  {
      // constrói um array de substituição com chaves em torno da chave de contexto.
      $replace = array();
      foreach ($context as $key => $val) {
          // verifica se o valor pode ser convertido para string
          if (!is_array($val) && (!is_object($val) || method_exists($val, '__toString'))) {
              $replace['{' . $key . '}'] = $val;
          }
      }

      // interpola os valores do array de substituição na mensagem e retorna
      return strtr($message, $replace);
  }

  // uma mensagem com espaço reservado delimitado por chaves
  $message = "User {username} created";
  
  // um array de contexto de nomes de espaços reservados => valores de substituição
  $context = array('username' => 'bolivar');

  // exibe "User bolivar created"
  echo interpolate($message, $context);
  ~~~

### 1.3 Contexto

- Todo método aceita um array como dados de contexto. Isso serve para assegurar
 qualquer informação estranha que não se ajuste bem a uma string. O array pode 
 conter qualquer coisa. Implementadores DEVEM garantir o tratamento dos dados de 
 contexto com a maior leniência possível. Um dado valor no contexto NÃO DEVE 
 lançar exceção nem, lançar qualquer erro, aviso ou notificação.

- Se um objeto ` Exception ` é passado no contexto de dados, ele DEVE estar na
 chave `'exception'`. As exceções de log são um padrão comum e isso permite que
 implementadores extraiam uma pilha de rastreiamento da exceção quando o log de
 backend suportar isto. Implementadores DEVEM ainda verificar se a chave 
 `'exception'` é, de fato, uma `Exception`, antes de usá-la como tal, pois esta
 PODE conter qualquer coisa.

### 1.4 Classes auxiliares e interfaces

- A classe `Psr\Log\AbstractLogger` permite você implementar a `LoggerInterface`
 muito facilmente, estendendo-a e implementando o método de ` log ` genérico.
 Os outros oito métodos encaminharão a mensagem e o contexto a ele.

- Similarmente, a utilização do `Psr\Log\LoggerTrait` requer apenas que você
 implemente um método `log` genérico. Note que como traits não podem implementar
 interfaces, neste caso você ainda terá que implementar a `LoggerInterface`.

- A `Psr\Log\NullLogger` é fornecida juntamente com a interface. Ela PODE ser
 usada por suários da interface para fornecer uma implementação alternativa de
 "buraco negro" se nenhum logger for dado a ele. Entretanto, o log condicional
 pode ser uma abordagem melhor se a criação de dados de contexto for cara.

- O `Psr\Log\LoggerAwareInterface` contém apenas o método
 `setLogger(LoggerInterface $logger)` e pode ser utilizado por frameworks para
 interligar automaticamente instâncias arbitrárias de um logger.

- O `Psr\Log\LoggerAwareTrait` trait pode ser utilizado para implementar a
 interface equivalente tão facilmente quanto qualquer outra classe. Isso dá a
 você o acesso a `$this->logger`.

- A classe `Psr\Log\LogLevel` mantém constantes para os oitros níveis de log.

## 2. Pacote

As interfaces e classes descritas como classes de exceção relevantes e uma
 suíte de teste para verificar sua implementação são disponibilizados como parte
 do pacote [psr/log](https://packagist.org/packages/psr/log).

## 3. `Psr\Log\LoggerInterface`

~~~php
<?php

namespace Psr\Log;

/**
 * Descreve uma instância de logger
 *
 * A mensagem DEVE ser uma string ou objeto implementando __toString().
 *
 * A mensagem PODE conter espaços reservados na forma: {foo} onde foo será 
 * substituído pelo dado de contexto presente na chave "foo".
 *
 * O array de contexto pode conter dados arbitrários, a única suposição que
 * pode ser feita pelos implementadores é que se uma instância de Exception for
 * dada para produzir uma pilha de rastréio, esta DEVE estar na chame nomeada
 * "exception".
 *
 * Ver https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-3-logger-interface.md
 * para a espeficicação completa da interface.
 */
interface LoggerInterface
{
    /**
     * Sistema está inutilizado.
     *
     * @param string $message
     * @param array $context
     * @return void
     */
    public function emergency($message, array $context = array());

    /**
     * Ação deve ser tomada imediatamente.
     *
     * Exemplo: Todo o website está fora do ar, banco de dados indisponível, etc.
     * Isso deve disparar os alertas SMS e te acordar.
     *
     * @param string $message
     * @param array $context
     * @return void
     */
    public function alert($message, array $context = array());

    /**
     * Condições críticas.
     *
     * Exemplo: Componente da aplicação indisponível, exceção não esperada.
     *
     * @param string $message
     * @param array $context
     * @return void
     */
    public function critical($message, array $context = array());

    /**
     * Erros em tempo de execução que não requerem ação imediata mas devem tipicamente
     * serem registrados e monitorados.
     *
     * @param string $message
     * @param array $context
     * @return void
     */
    public function error($message, array $context = array());

    /**
     * Ocorrências excepcionais que não sejam erros.
     *
     * Exemplo: Uso de APIs depreciadas, mal uso de uma API, coisas indesejadas
     * que não necessariamente estejam erradas.
     *
     * @param string $message
     * @param array $context
     * @return void
     */
    public function warning($message, array $context = array());

    /**
     * Eventos normais, porém insignificantes.
     *
     * @param string $message
     * @param array $context
     * @return void
     */
    public function notice($message, array $context = array());

    /**
     * Eventos interessantes.
     *
     * Exemplo: Logins de usuários, logs de SQL.
     *
     * @param string $message
     * @param array $context
     * @return void
     */
    public function info($message, array $context = array());

    /**
     * Informação detalhada para debug.
     *
     * @param string $message
     * @param array $context
     * @return void
     */
    public function debug($message, array $context = array());

    /**
     * Logs com um nível arbitrário.
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
 * Descreve uma instância de logger-aware.
 */
interface LoggerAwareInterface
{
    /**
     * Coloca uma instância de logger no objeto.
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
    const EMERGENCY = 'emergency';
    const ALERT     = 'alert';
    const CRITICAL  = 'critical';
    const ERROR     = 'error';
    const WARNING   = 'warning';
    const NOTICE    = 'notice';
    const INFO      = 'info';
    const DEBUG     = 'debug';
}
~~~

# Apache-RocketMQ
:zap: # Apache RocketMQ com Spring Boot

# 1. Introdução
Neste tutorial, criaremos um produtor e consumidor de mensagem usando Spring Boot e Apache RocketMQ, uma plataforma de transmissão de dados e mensagens distribuídas de código aberto.

# 2. Dependências
Para projetos Maven, precisamos adicionar a dependência RocketMQ Spring Boot Starter:
```
<dependência>
    <groupId> org.apache.rocketmq </groupId>
    <artifactId> rocketmq-spring-boot-starter </artifactId>
    <versão> 2.0.4 </version>
</dependency>
```

# 3. Produção de mensagens
Para nosso exemplo, criaremos um produtor de mensagem básico que enviará eventos sempre que o usuário adicionar ou remover um item do carrinho de compras.

Primeiro, vamos configurar a localização do nosso servidor e o nome do grupo em nosso application.properties:

rocketmq.name-server = 127.0.0.1: 9876
rocketmq.producer.group = cart-producer-group
Observe que se tivéssemos mais de um servidor de nomes, poderíamos listá-los como host: porta; host: porta.

Agora, para mantê-lo simples, vamos criar um aplicativo CommandLineRunner e gerar alguns eventos durante a inicialização do aplicativo:

```
@SpringBootApplication
public class CartEventProducer implementa CommandLineRunner {
 
    @Autowired
    private RocketMQTemplate rocketMQTemplate;
 
    public static void main (String [] args) {
        SpringApplication.run (CartEventProducer.class, args);
    }
 
    public void run (String ... args) throws Exception {
        rocketMQTemplate.convertAndSend ("cart-item-add-topic", novo CartItemEvent ("bike", 1));
        rocketMQTemplate.convertAndSend ("cart-item-add-topic", novo CartItemEvent ("computer", 2));
        rocketMQTemplate.convertAndSend ("cart-item-removed-topic", novo CartItemEvent ("bike", 1));
    }
}
```

O CartItemEvent consiste em apenas duas propriedades - o id do item e uma quantidade:

```
class CartItemEvent {
    private String itemId;
    quantidade int privada;
 
    // construtor, getters e setters
}
```
No exemplo acima, usamos o método convertAndSend (), um método genérico definido pela classe abstrata AbstractMessageSendingTemplate, para enviar nossos eventos de carrinho. Recebe dois parâmetros: um destino, que em nosso caso é um nome de tópico e uma carga útil da mensagem.

# 4. Mensagem ao consumidor
Consumir mensagens RocketMQ é tão simples quanto criar um componente Spring anotado com @RocketMQMessageListener e implementar a interface RocketMQListener:

```
@SpringBootApplication
public class CartEventConsumer {
 
    public static void main (String [] args) {
        SpringApplication.run (CartEventConsumer.class, args);
    }
 
    @Serviço
    @RocketMQMessageListener (
      topic = "cart-item-add-topic",
      consumerGroup = "cart-consumer_cart-item-add-topic"
    )
    public class CardItemAddConsumer implementa RocketMQListener <CartItemEvent> {
        public void onMessage (CartItemEvent addItemEvent) {
            log.info ("Adicionando item: {}", addItemEvent);
            // lógica adicional
        }
    }
 
    @Serviço
    @RocketMQMessageListener (
      topic = "cart-item-removed-topic",
      consumerGroup = "cart-consumer_cart-item-removed-topic"
    )
    public class CardItemRemoveConsumer implementa RocketMQListener <CartItemEvent> {
        public void onMessage (CartItemEvent removeItemEvent) {
            log.info ("Removendo item: {}", removeItemEvent);
            // lógica adicional
        }
    }
}
```

Precisamos criar um componente separado para cada tópico de mensagem que estamos ouvindo. Em cada um desses ouvintes, definimos o nome do tópico e o nome do grupo de consumidores por meio da anotação @RocketMQMessageListener.

# 5. Transmissão Síncrona e Assíncrona
Nos exemplos anteriores, usamos o método convertAndSend para enviar nossas mensagens. Porém, temos algumas outras opções.

Poderíamos, por exemplo, chamar syncSend, que é diferente de convertAndSend porque retorna o objeto SendResult.

Pode ser usado, por exemplo, para verificar se nossa mensagem foi enviada com sucesso ou obter seu id:

```
public void run (String ... args) throws Exception {
    SendResult addBikeResult = rocketMQTemplate.syncSend ("cart-item-add-topic",
      novo CartItemEvent ("bicicleta", 1));
    SendResult addComputerResult = rocketMQTemplate.syncSend ("cart-item-add-topic",
      novo CartItemEvent ("computador", 2));
    SendResult removeBikeResult = rocketMQTemplate.syncSend ("cart-item-removed-topic",
      novo CartItemEvent ("bicicleta", 1));
}
```
Como convertAndSend, esse método é retornado apenas quando o procedimento de envio é concluído.

Devemos usar a transmissão síncrona em casos que exigem alta confiabilidade, como mensagens de notificação importantes ou notificação por SMS.

Por outro lado, podemos querer enviar a mensagem de forma assíncrona e ser notificados quando o envio for concluído.

Podemos fazer isso com asyncSend, que usa SendCallback como parâmetro e retorna imediatamente:

```
rocketMQTemplate.asyncSend ("cart-item-add-topic", new CartItemEvent ("bike", 1), new SendCallback () {
    @Sobrepor
    public void onSuccess (SendResult sendResult) {
        log.error ("Item do carrinho enviado com sucesso");
    }
 
    @Sobrepor
    public void onException (Throwable throwable) {
        log.error ("Exceção durante o envio do item do carrinho", que pode ser jogado);
    }
});
```
Usamos transmissão assíncrona em casos que exigem alto rendimento.

Por último, para cenários em que temos requisitos de rendimento muito altos, podemos usar sendOneWay em vez de asyncSend. sendOneWay é diferente de asyncSend porque não garante que a mensagem seja enviada.

A transmissão unidirecional também pode ser usada para casos de confiabilidade comuns, como coleta de logs.


# 6. Envio de mensagens na transação
RocketMQ nos fornece a capacidade de enviar mensagens dentro de uma transação. Podemos fazer isso usando o método sendInTransaction ():

```
MessageBuilder.withPayload (new CartItemEvent ("bike", 1)). Build ();
rocketMQTemplate.sendMessageInTransaction ("transação de teste", "nome do tópico", msg, nulo);
Além disso, devemos implementar uma interface RocketMQLocalTransactionListener:

@RocketMQTransactionListener (txProducerGroup = "test-transaction")
class TransactionListenerImpl implementa RocketMQLocalTransactionListener {
      @Sobrepor
      public RocketMQLocalTransactionState executeLocalTransaction (mensagem msg, objeto arg) {
          // ... processo de transação local, retorna ROLLBACK, COMMIT ou UNKNOWN
          return RocketMQLocalTransactionState.UNKNOWN;
      }
 
      @Sobrepor
      public RocketMQLocalTransactionState checkLocalTransaction (mensagem msg) {
          // ... verificar o status da transação e retornar ROLLBACK, COMMIT ou UNKNOWN
          return RocketMQLocalTransactionState.COMMIT;
      }
}
```

Em sendMessageInTransaction (), o primeiro parâmetro é o nome da transação. Deve ser igual ao campo do membro txProducerGroup de @ RocketMQTransactionListener.


# 7. Configuração do Produtor de Mensagem
Também podemos configurar aspectos do próprio produtor da mensagem:

```
rocketmq.producer.send-message-timeout: O tempo limite de envio da mensagem em milissegundos - o valor padrão é 3000
rocketmq.producer.compress-message-body-threshold: Limite acima do qual RocketMQ compactará as mensagens - o valor padrão é 1024.
rocketmq.producer.max-message-size: O tamanho máximo da mensagem em bytes - o valor padrão é 4096.
rocketmq.producer.retry-times-when-send-async-failed: O número máximo de novas tentativas para executar internamente no modo assíncrono antes de enviar falha - o valor padrão é 2.
rocketmq.producer.retry-next-server: Indica se deve tentar novamente outro broker ao enviar falha internamente - o valor padrão é false.
rocketmq.producer.retry-times-when-send-failed: O número máximo de novas tentativas para executar internamente no modo assíncrono antes de enviar falha - o valor padrão é 2.
```
# 8. Conclusão
Neste artigo, aprendemos como enviar e consumir mensagens usando Apache RocketMQ e Spring Boot.

# 🏦 MicroRabbitMQ - Explicação Completa do Projeto

## 📋 Índice
1. [Visão Geral](#visão-geral)
2. [Arquitetura do Sistema](#arquitetura-do-sistema)
3. [Fluxo Completo da Mensageria](#fluxo-completo-da-mensageria)
4. [Estrutura por Camadas](#estrutura-por-camadas)
5. [Hierarquia de Classes](#hierarquia-de-classes)
6. [Como Executar](#como-executar)
7. [Conceitos Importantes](#conceitos-importantes)

---

## 🎯 Visão Geral

Este é um sistema de **microserviços bancários** que usa **RabbitMQ** para comunicação assíncrona e **CQRS** (Command Query Responsibility Segregation) com **MediatR**.

### Componentes Principais:
- **Banking API** (Port 5000): API REST para operações bancárias
- **RabbitMQ** (Port 5672): Message broker para comunicação assíncrona
- **SQL Server** (Port 1433): Banco de dados para persistência

---

## 🏗️ Arquitetura do Sistema

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Banking API   │    │   RabbitMQ      │    │   SQL Server    │
│   (Port 5000)   │    │   (Port 5672)   │    │   (Port 1433)   │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

---

## 📋 Fluxo Completo da Mensageria

### **1. Cliente faz uma requisição POST**
```http
POST /api/banking
Content-Type: application/json

{
  "accountFrom": 1,
  "toAccount": 2, 
  "transferAmount": 100.00
}
```

### **2. BankingController recebe a requisição**
```csharp
// BankingController.cs - linha 28-33
[HttpPost]
public IActionResult Post([FromBody] AccountTransfer accountTransfer)
{
    _accountService.Transfer(accountTransfer);
    return Ok(accountTransfer);
}
```

### **3. AccountService processa a transferência**
```csharp
// AccountService.cs - linha 31-40
public void Transfer(AccountTransfer accountTransfer)
{
    var createTransferCommand = new CreateTransferCommand(
        accountTransfer.AccountFrom,
        accountTransfer.ToAccount,
        accountTransfer.TransferAmount
    );
    
    _bus.SendCommand(createTransferCommand); // 📤 Envia comando
}
```

### **4. RabbitMQQBus processa o comando**
```csharp
// RabbitMQQBus.cs - linha 31-34
public Task SendCommand<T>(T command) where T : Command
{
    return _mediator.Send(command); // 📨 MediatR roteia o comando
}
```

### **5. TransferCommandHandler executa o comando**
```csharp
// TransferCommandHandler.cs - linha 22-27
public Task<bool> Handle(CreateTransferCommand request, CancellationToken cancellationToken)
{
    // 📢 Publica evento no RabbitMQ
    _bus.Publish(new TransferCreatedEvent(request.From, request.To, request.Amout));
    return Task.FromResult(true);
}
```

### **6. RabbitMQQBus publica o evento no RabbitMQ**
```csharp
// RabbitMQQBus.cs - linha 35-56
public async void Publish<T>(T @event) where T : Event
{
    var factory = new ConnectionFactory() { HostName = "localhost" };
    using var connection = await factory.CreateConnectionAsync();
    using var channel = await connection.CreateChannelAsync();
    
    var eventName = @event.GetType().Name; // "TransferCreatedEvent"
    channel.QueueDeclareAsync(eventName, false, false);
    
    var message = JsonConvert.SerializeObject(@event);
    var body = Encoding.UTF8.GetBytes(message);
    
    await channel.BasicPublishAsync("", eventName, body); // 🚀 Publica no RabbitMQ
}
```

### **7. RabbitMQ armazena a mensagem**
- **Queue**: `TransferCreatedEvent`
- **Message**: JSON serializado do evento
- **Routing**: Baseado no nome do evento

### **8. Outros microserviços podem consumir o evento**
```csharp
// Quando um microserviço se inscreve:
public void Subscribe<T, TH>() where T : Event where TH : IEventHandler<T>
{
    // Registra o handler para o evento
    // Inicia o consumo da fila
    StartBasicConsume<T>();
}
```

---

## 🔄 Hierarquia de Classes

```
Message (base)
├── Command (abstrata)
│   └── TransferCommand (abstrata)
│       └── CreateTransferCommand (concreta)
└── Event (abstrata)
    └── TransferCreatedEvent (concreta)
```

### **Classes Principais:**

#### **Message** (`MicroRabbit.Domain.Core/Events/Message.cs`)
```csharp
public abstract class Message : IRequest<bool>
{
    public string MessageType { get; protected set; }
    
    protected Message() 
    {
        MessageType = GetType().Name;
    }
}
```

#### **Command** (`MicroRabbit.Domain.Core/Commands/Command.cs`)
```csharp
public abstract class Command : Message
{
    public DateTime Timestamp { get; protected set; }
    
    protected Command()
    {
        Timestamp = DateTime.Now;
    }
}
```

#### **TransferCommand** (`MicroRabbit.Banking.Domain/Commands/TransferCommand.cs`)
```csharp
public abstract class TransferCommand : Command
{
    public int From { get; protected set; }
    public int To { get; protected set; }
    public decimal Amout { get; protected set; }
}
```

#### **CreateTransferCommand** (`MicroRabbit.Banking.Domain/Commands/CreateTransferCommand.cs`)
```csharp
public class CreateTransferCommand : TransferCommand
{
    public CreateTransferCommand(int from, int to, decimal amout)
    {
        From = from;
        To = to;
        Amout = amout;
    }
}
```

#### **Event** (`MicroRabbit.Domain.Core/Events/Event.cs`)
```csharp
public abstract class Event
{
    public DateTime Timestamp { get; protected set; }
    
    protected Event()
    {
        Timestamp = DateTime.Now;
    }
}
```

#### **TransferCreatedEvent** (`MicroRabbit.Banking.Domain/Events/TransferCreatedEvent.cs`)
```csharp
public class TransferCreatedEvent : Event
{
    public int From { get; private set; }
    public int To { get; private set; }
    public decimal Amount { get; private set; }
    
    public TransferCreatedEvent(int from, int to, decimal amount)
    {
        From = from;
        To = to;
        Amount = amount;
    }
}
```

---

## 📁 Estrutura por Camadas

### **API Layer** (`MicroRabbit.Banking.Api`)
- **BankingController.cs**: Recebe requisições HTTP
- **AccountTransfer.cs**: Modelo de entrada para transferências
- **Program.cs**: Configuração da aplicação

### **Application Layer** (`MicroRabbit.Banking.Application`)
- **AccountService.cs**: Orquestra a lógica de negócio
- **IAccountService.cs**: Interface do serviço
- **AccountTransfer.cs**: Modelo de transferência

### **Domain Layer** (`MicroRabbit.Banking.Domain`)
- **Commands/**: `CreateTransferCommand.cs`, `TransferCommand.cs`
- **Events/**: `TransferCreatedEvent.cs`
- **CommandHandlers/**: `TransferCommandHandler.cs`
- **Models/**: `Account.cs`
- **Interfaces/**: `IAccountRepository.cs`

### **Data Layer** (`MicroRabbit.Banking.Data`)
- **Context/**: `BankingDbContext.cs`
- **Repository/**: `AccountRepository.cs`
- **Migrations/**: Migrações do Entity Framework

### **Infrastructure Layer** (`MicroRabbit.Infra.Bus`)
- **RabbitMQQBus.cs**: Implementação do barramento de eventos

### **Core Layer** (`MicroRabbit.Domain.Core`)
- **Bus/**: `IEventBus.cs`, `IEventHandler.cs`
- **Commands/**: `Command.cs`
- **Events/**: `Event.cs`, `Message.cs`

### **IoC Layer** (`MicroRabbit.IoC`)
- **DependencyContainer.cs**: Injeção de dependências

---

## 🚀 Como Executar

### **1. Subir os containers**
```bash
# Navegue até a pasta do projeto
cd MicroRabbitMQ

# Execute o docker-compose
docker-compose up -d
```

### **2. Verificar se os serviços estão rodando**
```bash
# Verificar containers ativos
docker ps

# Verificar logs
docker-compose logs -f
```

### **3. Fazer uma requisição de teste**
```bash
# Usando curl
curl -X POST http://localhost:5000/api/banking \
  -H "Content-Type: application/json" \
  -d '{
    "accountFrom": 1,
    "toAccount": 2,
    "transferAmount": 100.00
  }'

# Ou usando PowerShell
Invoke-RestMethod -Uri "http://localhost:5000/api/banking" \
  -Method POST \
  -ContentType "application/json" \
  -Body '{
    "accountFrom": 1,
    "toAccount": 2,
    "transferAmount": 100.00
  }'
```

### **4. Verificar no RabbitMQ Management**
- **URL**: http://localhost:15672
- **Usuário**: `guest`
- **Senha**: `guest`
- **Verificar**: Fila `TransferCreatedEvent`

### **5. Verificar no SQL Server**
- **Server**: localhost,1433
- **Database**: BankingDB
- **User**: sa
- **Password**: YourStrong@Passw0rd

---

## 🔑 Conceitos Importantes

### **CQRS (Command Query Responsibility Segregation)**
- **Commands**: Operações de escrita (CreateTransferCommand)
- **Queries**: Operações de leitura (GetAccounts)
- **Separação**: Diferentes modelos para leitura e escrita

### **Event-Driven Architecture**
- **Eventos**: Representam algo que aconteceu no sistema
- **Reativo**: Sistema responde a eventos
- **Desacoplado**: Serviços não dependem diretamente uns dos outros

### **MediatR Pattern**
- **Mediator**: Padrão que desacopla objetos
- **Send**: Para commands
- **Publish**: Para events
- **Handlers**: Processam commands e events

### **RabbitMQ**
- **Message Broker**: Intermediário para mensagens
- **Queues**: Filas para armazenar mensagens
- **Publish/Subscribe**: Padrão de comunicação
- **Durabilidade**: Mensagens persistem mesmo se o consumidor estiver offline

### **Docker & Docker Compose**
- **Containerização**: Isolamento de aplicações
- **Orquestração**: Múltiplos serviços em conjunto
- **Networking**: Comunicação entre containers

---

## 📊 Resumo do Fluxo

```
1. Cliente → POST para API
2. Controller → Chama AccountService
3. Service → Cria CreateTransferCommand
4. Bus → Roteia via MediatR
5. Handler → Processa comando e publica evento
6. RabbitMQ → Armazena TransferCreatedEvent
7. Outros serviços → Podem consumir o evento
```

---

## 🛠️ Arquivos de Configuração

### **docker-compose.yml**
- Configuração dos serviços (API, RabbitMQ, SQL Server)
- Redes e volumes
- Variáveis de ambiente

### **Dockerfile**
- Configuração da imagem da API
- Build e runtime da aplicação

### **appsettings.json**
- Configurações da aplicação
- Connection strings
- Configurações do RabbitMQ

---

## 📝 Próximos Passos

1. **Implementar Event Handlers** para processar os eventos
2. **Adicionar validações** nos commands
3. **Implementar logging** e monitoramento
4. **Adicionar testes** unitários e de integração
5. **Implementar outros microserviços** (Transfer, Notification, etc.)

---

## 🔗 Links Úteis

- [RabbitMQ Documentation](https://www.rabbitmq.com/documentation.html)
- [MediatR Documentation](https://github.com/jbogard/MediatR)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [CQRS Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/cqrs)

---

**Data de Criação**: $(Get-Date)
**Versão**: 1.0
**Autor**: Explicação detalhada do projeto MicroRabbitMQ


# TRF3 Events

A aplicação TRF3 Events é um sistema automatizado de busca e processamento de eventos de processos do Tribunal Regional Federal da 3ª Região, que extrai dados processuais, documenta eventos e realiza identificação de participantes com enriquecimento de CPF através de múltiplos providers.

## **Pré-requisitos**

- Node.js versão 20.x

## Setup

1. **Criar arquivo .npmrc**
   > [Gere o token em Settings → Developer settings → Personal access tokens → Tokens (classic)](https://github.com/settings/tokens)
   >
   > Substitua o token gerado em TOKEN

```bash
@precato:registry=https://npm.pkg.github.com/
@precatoorg:registry=https://npm.pkg.github.com/
//npm.pkg.github.com/:_authToken=TOKEN
```

2. **Instalar dependências**

   ```bash
   npm install
   ```

3. **Criar arquivo .env**

> Copie as variáveis de ambiente do arquivo _.env.example_

4. **Iniciar containers necessários**
   - Ligar container do _rabbit_
   - Ligar a aplicação do _database-server_

5. **Executar aplicação**
   ```bash
   npm run dev
   ```

```mermaid
%% Fluxo: Busca de Eventos de Processos (UPDATE_TRF3_PROCESS_EVENTS)

sequenceDiagram

    participant UPDATE_TRF3_PROCESS_EVENTS
    participant searchProcessEventsConsumer
    participant SearchProcessEventsService
    participant Trf3PjeProvider
    participant postgres

    %% 1) Mensagem chega na fila e é consumida

    Note over UPDATE_TRF3_PROCESS_EVENTS: Mensagem aguardando consumo
    rect rgb(64, 128, 128)
        UPDATE_TRF3_PROCESS_EVENTS-->>searchProcessEventsConsumer: JSON { processNumber: string }
    end

    %% 2) searchProcessEventsConsumer chama o serviço principal
    rect rgb(128, 64, 128)
        searchProcessEventsConsumer->>SearchProcessEventsService: SearchProcessEventsService.execute({ processNumber })
    end

    %% 3) Serviço busca dados do processo no TRF3 (PJe)
    rect rgb(128, 160, 128)
        SearchProcessEventsService->>Trf3PjeProvider: Trf3PjeProvider.getProcessData({ processNumber })
        Trf3PjeProvider-->>SearchProcessEventsService: [ processData: {...}, process: {...} ]
    end

    %% 4) Verificar eventos já existentes
    rect rgb(128, 128, 160)
        SearchProcessEventsService->>postgres: findManyByProcessId<br/>(Busca dados já registrados)
        postgres-->>SearchProcessEventsService: processEvents<br/>(eventos já existentes)
    end

    %% 5) Registrar novos eventos
    rect rgb(128, 128, 160)
        SearchProcessEventsService->>postgres: createMany<br/>(registrar novos eventos)
        postgres-->>SearchProcessEventsService: ok
    end

    %% 6) Fim
    SearchProcessEventsService-->>searchProcessEventsConsumer: ok
    rect rgb(32, 32, 128)
        searchProcessEventsConsumer-->>UPDATE_TRF3_PROCESS_EVENTS: ACK<br/>(Fila pode remover a mensagem)
    end
```

### Fluxo: Identificação de Participantes de Processo

```mermaid
%% Fluxo: Identificação de Participantes de Processo com Fallbacks

sequenceDiagram
    participant SEARCH_PROCESS_PARTICIPANT_IDENTIFICATIONS
    participant Consumer
    participant SearchProcessParticipantIdentificationsService
    participant escavadorProcessSearchProvider
    participant novaVidaEnrichmentProvider
    participant Trf3ProcessProvider
    participant trf3HttpSenderProvider
    participant PostgresDB

    %% ETAPA 1: Recebimento e disparo do serviço
    rect rgb(64, 128, 128)
        SEARCH_PROCESS_PARTICIPANT_IDENTIFICATIONS-->>Consumer: { process_number, active_pole }
    end
    rect rgb(128, 64, 128)
        Consumer->>SearchProcessParticipantIdentificationsService: { process_number, active_pole }<br/>active_pole = [nome completo - 123.XXX.XXX-XX]<br/>CPF somente com os 3 primeiros digitos
    end

    %% ETAPA 2: Primeira tentativa - Escavador
    rect rgb(128, 160, 128)
        SearchProcessParticipantIdentificationsService->>escavadorProcessSearchProvider: Busca Escavador<br/>(process_number)
        escavadorProcessSearchProvider-->>SearchProcessParticipantIdentificationsService: CPFs encontrados
    end
    alt Encontrados CPFSs
    rect rgb(128, 128, 128)
        SearchProcessParticipantIdentificationsService->>trf3HttpSenderProvider: { process_number: "", cpf: "" }
        trf3HttpSenderProvider-->>SearchProcessParticipantIdentificationsService: ok
    %% ETAPA 3: Verificação de lacunas (nomes sem CPF no DB)
    end
    else Falta CPFs
    rect rgb(128, 128, 160)
        SearchProcessParticipantIdentificationsService->>PostgresDB: findManyByProcessoOriginario<br/>(verifica nomes banco)
        PostgresDB-->>SearchProcessParticipantIdentificationsService: processos existentes<br/>(nomes faltantes precisam ser enriquecidos)
    end
    %% ETAPA 4: Segunda tentativa - Enriquecimento Nova Vida
    else Falta Nomes - Enriquecimento com Nova Vida
        rect rgb(128, 160, 128)
        SearchProcessParticipantIdentificationsService->>novaVidaEnrichmentProvider: getCpfsByFullName<br/>(busca CPF por nome completo)
        novaVidaEnrichmentProvider-->>SearchProcessParticipantIdentificationsService: CPFs encontrados
    end
    else Encontrados CPFs
    rect rgb(128, 128, 128)
        SearchProcessParticipantIdentificationsService->>trf3HttpSenderProvider: { process_number: "", cpf: "" }
        trf3HttpSenderProvider-->>SearchProcessParticipantIdentificationsService: ok
    end
    %% ETAPA 5: Terceira tentativa - Busca logada TRF3 (fallback final)
    else Falta Nomes - Busca Logada
    rect rgb(128, 160, 128)
        SearchProcessParticipantIdentificationsService->>Trf3ProcessProvider: getParticipantIdentifications<br/>(busca logada direta)
        Trf3ProcessProvider-->>SearchProcessParticipantIdentificationsService: Retorna Todos os CPFs
    end
    else Encontrados CPFs
    rect rgb(128, 128, 128)
        SearchProcessParticipantIdentificationsService->>trf3HttpSenderProvider: { process_number: "", cpf: "" }
        trf3HttpSenderProvider-->>SearchProcessParticipantIdentificationsService: ok
    end
    end

    %% ETAPA 7: Desfecho (sucesso ou erro)
    alt Nenhuma tentativa funcionou
    rect rgb(128, 32, 32)
        SearchProcessParticipantIdentificationsService-->>Consumer: Erro: Não foi possível identificar todos os CPFs
    end
    else Enriquecimento concluído
    rect rgb(128, 32, 256)
        SearchProcessParticipantIdentificationsService-->>Consumer: Processamento concluído
    end
    end

    %% ETAPA 8: Finalização da mensagem
    rect rgb(32, 32, 128)
        Consumer-->>SEARCH_PROCESS_PARTICIPANT_IDENTIFICATIONS: ACK<br/>(Fila pode remover a mensagem)
    end
```

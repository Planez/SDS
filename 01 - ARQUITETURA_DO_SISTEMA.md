# ARQUITETURA TÉCNICA DO FRAMEWORK SDS

## VISÃO GERAL DA ARQUITETURA E TOPOLOGIA
Descrição: Definição da topologia macro e do modelo arquitetural do framework SDS
VALOR PARA A IA:
- Separação de responsabilidades: Orienta a manutenção de limites claros entre os componentes, sem tratar como aprovados os caminhos de comunicação ainda pendentes.
- Roteamento de metadados: Registra como proposta a leitura de metadados de backend no Oracle e de metadados de interface via Redis; o fallback e a sincronização precisam de decisão arquitetural.
- Exposição de APIs: Registra como proposta o ORDS para clientes externos, sem concluir que esse é o único canal até a stack e os fluxos serem aprovados.

O texto para este tópico é:
A arquitetura do SDS será implantada na **Oracle Cloud Infrastructure (OCI)**, conforme decisão confirmada. O modelo **Metadata-Driven Architecture** é o princípio estrutural do framework. Os componentes abaixo descrevem a arquitetura proposta, ainda sujeita à validação da stack e dos fluxos.

1. **Núcleo Central de Dados, Metadados e Backend (Oracle Database 26ai + PL/SQL):**
   * É proposto como fonte de verdade para os dados de negócio e metadados (contratos JSON/relacionais), com garantias transacionais e de segurança a definir.
   * Todo o motor de execução de backend reside em pacotes **PL/SQL**, que leem os metadados diretamente das tabelas internas do Oracle em memória/buffer cache, sem depender de camadas externas ou Redis para processar regras e cargas em lote (*set-based*).
2. **Camada de Exposição de APIs para Clientes Finais (ORDS):**
   * O **Oracle REST Data Services (ORDS)** atua exclusivamente como porta de entrada REST para **clientes finais e integradores externos**, conectando-se diretamente aos pacotes PL/SQL e ao Oracle Database (`Cliente Final -> ORDS -> PL/SQL -> Oracle`), eliminando servidores intermediários de aplicação no tráfego de APIs de consumo.
3. **Motor de Interface Web e Cache Sincronizado (Django + Redis):**
   * O **Django** é proposto como motor de renderização dinâmica. O Redis será usado para contratos de tela e eventual fallback ao Oracle são decisões de fluxo ainda pendentes
   * A sincronização de metadados entre Oracle e Redis está proposta, mas o mecanismo, o tratamento de falhas e a garantia de consistência ainda precisam ser definidos.
4. **Camada de Automação e Processamento Assíncrono (Python + Celery):**
   * Orquestra rotinas externas (ingestão de arquivos, scrapers, integrações de nuvem OCI) e tarefas pesadas de longa duração em segundo plano (como geração de embeddings vetoriais), acionando os motores PL/SQL de forma assíncrona via filas gerenciadas pelo Celery/Redis.

## PRINCÍPIOS DE ORIENTAÇÃO A OBJETOS E ENGENHARIA DE CLASSES
Descrição: Diretrizes para estruturar os motores do SDS por meio de classes com responsabilidades claras, independentes e verificáveis, usando SOLID como base de projeto.

VALOR PARA A IA:
- Aplicar os princípios SOLID para orientar coesão, extensão, substituição e dependências entre classes e tipos.
- Projetar componentes com responsabilidades delimitadas, evitando classes que concentrem responsabilidades sem relação entre si.
- Facilitar testes unitários isolados e a automação da suíte, reduzindo dependências diretas de banco, cache, rede, relógio e sistema de arquivos.
- Manter a arquitetura orientada a metadados: as classes implementam mecanismos estáveis de interpretação e execução; regras variáveis de negócio permanecem nos contratos de metadados aprovados.

### Aplicação de SOLID

- **S — Single Responsibility Principle:** cada classe ou tipo deve ter uma responsabilidade coesa e motivos de mudança relacionados.
- **O — Open/Closed Principle:** permitir extensão de comportamento sem modificar mecanismos estáveis, preferindo contratos e composição quando apropriado.
- **L — Liskov Substitution Principle:** implementações derivadas ou alternativas devem respeitar o contrato do tipo que substituem, sem alterar as expectativas de quem o utiliza.
- **I — Interface Segregation Principle:** manter contratos pequenos e específicos para que cada consumidor dependa somente das operações de que necessita.
- **D — Dependency Inversion Principle:** separar políticas de alto nível dos detalhes de infraestrutura; depender de contratos e fornecer dependências pelas fronteiras dos componentes, favorecendo substituição em testes.

Diretrizes de engenharia:
- Adotar orientação a objetos forte em todas as camadas cuja tecnologia ofereça suporte ao paradigma, incluindo os motores e serviços Python e o backend PL/SQL.
- No PL/SQL, utilizar fortemente tipos de objeto (`CREATE TYPE`) e seus métodos, implementados em corpos de tipo (`CREATE TYPE BODY`), como estrutura central de engenharia de classes. Modelar estado e comportamento coesos no tipo; usar métodos de instância ou estáticos de acordo com a responsabilidade, sem reduzir a orientação a objetos a pacotes com procedimentos.
- Manter cada classe ou tipo de objeto focado em responsabilidades coesas, com interfaces explícitas e dependências controladas, evitando acoplamento oculto a estado global ou infraestrutura.
- Projetar o comportamento das classes, tipos e métodos para permitir testes unitários isolados e automatizados. Substituir ou controlar dependências externas nos testes quando possível; testes de integração complementam, mas não substituem, os testes isolados.
- Utilizar pacotes PL/SQL quando forem adequados para agrupar operações relacionadas, APIs de módulo ou serviços auxiliares; eles complementam, mas não substituem, os tipos de objeto e seus métodos como mecanismo de orientação a objetos.
- Aplicar SOLID como guia de projeto, não como obrigação de criar abstrações artificiais. Respeitar as capacidades e limitações de cada linguagem, mantendo os mesmos objetivos de coesão, contratos claros, substituibilidade e baixo acoplamento.
- Definir a ferramenta e a estratégia de testes automatizados para tipos e métodos PL/SQL conforme as capacidades do ambiente Oracle, preservando o requisito de testabilidade independente.

## STACK TECNOLÓGICA PROPOSTA E DIVISÃO DE RESPONSABILIDADES
Descrição: Registro da stack considerada para o framework SDS na OCI e das responsabilidades pretendidas para seus componentes. Tecnologias e versões desta seção estão confirmadas.
VALOR PARA A IA:
- Registrar a arquitetura tecnológica em avaliação e evitar tratar tecnologias ou versões não aprovadas como restrições imutáveis.
- Orientar a discussão sobre responsabilidades e fronteiras entre componentes.

O texto para este tópico é:

| Camada / Função | Tecnologia proposta (a validar) | Responsabilidade pretendida e limites de atuação |
| :--- | :--- | :--- |
| **Infraestrutura Cloud** | **Oracle Cloud Infrastructure (OCI)** | Ambiente oficial de hospedagem, rede, armazenamento de objetos (OCI Object Storage) e segurança de infraestrutura do ecossistema SDS. |
| **Banco de Dados & Busca Semântica** | **Oracle Database 26ai** | Custódia centralizada e segura dos metadados do framework (tabelas de configuração JSON/relacionais) e dos dados de negócio. Execução obrigatória de consultas híbridas combinando filtros relacionais com **Oracle AI Vector Search** (busca vetorial semântica nativa). |
| **Motor de Backend** | **Oracle PL/SQL** | Núcleo proposto de execução da lógica e processamento de dados. Tipos de objeto e métodos PL/SQL são o mecanismo previsto para engenharia de classes; pacotes podem complementar essa organização. Lê metadados de backend no banco e pode expor operações para ORDS e Python/Celery, conforme fluxos aprovados. |
| **APIs para Clientes Finais** | **Oracle REST Data Services (ORDS)** | Camada nativa de exposição de serviços REST destinada exclusivamente ao consumo de dados por **clientes finais e sistemas externos**, conectando-se diretamente às rotinas PL/SQL no Oracle. Não é utilizado pelo frontend Django. |
| **Motor de Frontend (Web)** | **Python / Django** | Interpretador de interfaces SDUI. Responsável pela gestão de sessão web, validação de contratos de tela e renderização dinâmica de templates a partir dos metadados lidos prioritariamente do Redis. |
| **Cache de Metadados & Broker** | **Redis** | Armazenamento em memória de alta velocidade para os metadados de interface consumidos pelo Django (mantido sincronizado via triggers do Oracle) e *message broker* para as filas de processamento do Celery. |
| **Automações e Filas** | **Python + Celery** | Desenvolvimento de automações externas, ingestão de arquivos (scrapers/parsing) e orquestração de tarefas assíncronas pesadas em segundo plano, invocando os motores PL/SQL sem bloquear requisições web. |

### Regras de Fronteira entre Tecnologias:
As fronteiras abaixo refletem a proposta atual e só devem ser tratadas como restrições de implementação após a aprovação da stack e dos fluxos:
1. **Metadados de backend:** A proposta é que o motor PL/SQL consulte seus metadados no Oracle, sem depender do Redis.
2. **Metadados de interface:** Oracle como fonte e Redis como cache são propostos; mecanismo de sincronização, consistência e tratamento de falhas permanecem pendentes.
3. **ORDS e acesso do Django:** O uso do ORDS para integrações externas e o canal do Django para ações e consultas precisam ser confirmados. Os trechos atuais divergem entre acesso direto ao Oracle e uso de ORDS.
4. **Busca semântica:** Oracle AI Vector Search é uma opção descrita para a stack proposta. Sua adoção obrigatória depende da aprovação do banco e dos requisitos de busca.

## FLUXOS DE EXECUÇÃO DOS MOTORES E A HOMEOSTASE SISTÊMICA (RUNTIME EXECUTION FLOWS)
Descrição: Fluxos operacionais propostos para o ecossistema SDS. Resiliência, idempotência e observabilidade são objetivos de arquitetura; seus mecanismos e critérios de aceitação precisam ser definidos antes de serem tratados como garantias.
VALOR PARA A IA:
- Documentar os fluxos pretendidos sem converter metas de resiliência em garantias não verificadas.
- Exigir que idempotência, retomada, telemetria e alertas tenham comportamento e critérios de validação definidos.

O texto para este tópico é:

O desenho atual apresenta três fluxos pretendidos. Sua independência, resiliência e comportamento de recuperação ainda dependem de validação e de critérios de aceitação:

### 1. Fluxo de Renderização de Interface e Resiliência de Frontend (SDUI)
Este fluxo descreve a entrega pretendida de telas e formulários dinâmicos via Django. O comportamento em caso de indisponibilidade do cache ou do Oracle ainda precisa ser especificado:

1. **Requisição e Rastreabilidade Contextual:** O usuário acessa uma rota no navegador; o **Django** intercepta a chamada e propaga um **Correlation ID** unificado.
2. **Leitura de Contrato e Telemetria Homeostática:** O Django busca o metadado estruturado no **Redis**:
   * **Equilíbrio (Cache Hit):** Contrato recuperado em sub-milissegundo com incremento da métrica de sucesso (`sds_cache_hit_total`).
   * **Perturbação Controlada (Cache Miss):** Caso a chave esteja ausente, o sistema ativa seus mecanismos homeostáticos para proteger o organismo:
     * **Prioridade Absoluta à Experiência do Cliente (Zero Bloqueio):** O Django consulta diretamente as tabelas relacionais no Oracle **apenas o estritamente necessário para entregar a resposta imediata ao usuário**, blindando a experiência de qualquer latência de infraestrutura.
     * **Alerta Precoce (Nocicepção):** Incrementa a métrica de falha (`sds_cache_miss_total`) e emite um **Log Estruturado de Alerta** com o Correlation ID. (Taxas anormais de *miss* disparam alertas automáticos de desvio de padrão na infraestrutura).
     * **Auto-Cura Assíncrona (Self-Healing):** Paralelamente e em segundo plano, uma task assíncrona extrai o contrato do Oracle e repopula o Redis, reestabelecendo o equilíbrio térmico do cache para as requisições seguintes sem que o cliente atual espere.
3. **Validação e Renderização Dinâmica:** O motor Django valida o contrato relacional e renderiza os componentes Server-Driven UI.
4. **Submissão e Continuidade:** O formulário preenchido é despachado com o Correlation ID para o backend, garantindo rastreabilidade fechada.


### 2. Fluxo de Processamento de Backend, Cargas em Lote e Retomada (PL/SQL)
Este fluxo descreve o processamento transacional e em lote proposto para o Oracle Database. A estratégia de retomada e as garantias de idempotência devem ser definidas por processo:

1. **Acionamento com Propagação de Contexto:** Um processo em lote é disparado (via agendamento, Celery ou ORDS). O Correlation ID é injetado no contexto da sessão Oracle (`DBMS_APPLICATION_INFO`).
2. **Rastreabilidade:** A proposta registra o início e o resultado da execução. O uso de transação autônoma, estados e garantias de persistência precisa ser validado para cada tipo de operação.
3. **Processamento Set-Based Idempotente (Checkpoints):** 
   * O PL/SQL executa a lógica de negócio aproveitando o buffer cache do Oracle.
   * **Retomada e idempotência:** Checkpoints e retomada são objetivos propostos. Definir granularidade, semântica transacional e tratamento de efeitos externos; não assumir retomada exata nem ausência de reprocessamento sem esses critérios.
4. **Fechamento de Ciclo, Métricas e Alerta Proativo:** 
   * A proposta consolida os resultados e registra o estado da execução; estados, métricas e limites de auditoria ainda precisam ser definidos.
   * **Métricas e alertas:** Definir limiares, canal de notificação e objetivos de tempo de detecção e resposta; não há garantia de alerta antes de impacto ao cliente.

### 3. Fluxo de Sincronia Reativa de Metadados (Oracle $\rightarrow$ Redis)
Para manter o alinhamento interno entre a fonte da verdade e a camada de aceleração:

1. **Alteração Controlada:** Um metadado relacional é atualizado por um administrador no Oracle Database.
2. **Disparo de Gatilho Homeostático:** A **Trigger** no data source é acionada instantaneamente.
3. **Atualização ou invalidação do cache:** Definir como a mudança chega ao Redis, como são tratadas falhas e retries e qual janela de inconsistência é aceitável.

## SEGURANÇA, PROTEÇÃO DE METADADOS E OBSERVABILIDADE
Descrição: Diretrizes propostas para proteção de dados e metadados, gestão de privilégios e monitoramento. Os controles e garantias dependem de análise de riscos, requisitos regulatórios e validação técnica.
VALOR PARA A IA:
- Validar schemas e parâmetros e evitar execução de código ou SQL dinâmico não controlado.
- Tratar confidencialidade, criptografia, privilégio mínimo e dados pessoais como requisitos a definir e validar, sem assumir que uma técnica isolada garante conformidade.
- Definir como a rastreabilidade e a observabilidade serão implementadas em cada camada, respeitando a minimização e a proteção de dados nos logs.

O texto para este tópico é:

Como os metadados e os motores determinam o comportamento do ecossistema SDS, segurança e observabilidade devem ser consideradas preocupações transversais da arquitetura.

### Proteção de Dados e Metadados
Os controles específicos de proteção devem ser definidos conforme classificação dos dados, análise de ameaças e obrigações legais aplicáveis:

* **Classificação e minimização:** Identificar dados sensíveis e registrar apenas o necessário em armazenamento, tráfego e logs.
* **Criptografia e gestão de chaves:** Definir requisitos de criptografia em repouso e em trânsito, protocolos suportados e responsabilidades pela gestão de chaves. A menção a TDE e TLS na proposta de stack não é, por si só, validação de configuração ou cobertura completa.
* **Ofuscação e pseudonimização:** Avaliar caso a caso se são necessárias e adequadas. Não presumir que scramble ou ofuscação garantem conformidade com a LGPD ou impedem acesso por contas privilegiadas.
* **Execução segura de contratos:** Validar estruturas e parâmetros conforme schemas aprovados; não interpretar metadados como código executável arbitrário.
* **Controle de acesso:** Definir papéis, privilégios mínimos, segregação de funções e auditoria. Criptografia em repouso não substitui controles de acesso nem elimina, por si só, riscos de usuários privilegiados.

### Observabilidade Proativa e Rastreabilidade de Ponta a Ponta
Os mecanismos de observabilidade e seus objetivos de serviço ainda precisam de definição:

* **Rastreabilidade:** Correlation ID é uma proposta para correlacionar requisições e tarefas. Definir formato, propagação e regras para evitar inclusão de dados sensíveis.
* **Auditoria:** Definir eventos auditáveis, retenção e limites transacionais. O uso de `PRAGMA AUTONOMOUS_TRANSACTION` requer avaliação específica e não deve ser adotado automaticamente para todas as rotinas.
* **Métricas:** Cache hit/miss, latência, volumetria e rejeições são indicadores sugeridos; selecionar métricas, cardinalidade e limites após definir requisitos de operação.
* **Alertas:** Definir limiares, destinatários, canais e tempos de resposta. Não assumir detecção ou notificação antes de qualquer impacto ao usuário.

## PENDÊNCIAS DE VALIDAÇÃO ARQUITETURAL

Os itens abaixo precisam de decisão ou especificação antes de orientar a implementação:

- **Stack e versões:** a OCI,  Oracle Database 26ai, PL/SQL, Django, Redis, Celery, ORDS e demais tecnologias são escolhas aprovadas.
- **Testes de tipos e métodos PL/SQL:** a orientação a objetos forte por meio de tipos de objeto e métodos PL/SQL está definida como diretriz. Definir a ferramenta e a estratégia concreta para executar testes automatizados e isolados desses tipos e métodos.
- **Acesso do Django ao Oracle:** escolher o canal oficial e alinhar a regra de não utilizar ORDS internamente com a árvore de diretórios e os fluxos.
- **Sincronização Oracle–Redis:** escolher mecanismo, consistência, retries, comportamento em indisponibilidade e eventual fallback.
- **Resiliência e processamento:** definir semântica de idempotência, checkpoints, efeitos externos e retomada por tipo de processo.
- **Segurança e privacidade:** classificar dados, modelar ameaças e validar controles, incluindo requisitos legais aplicáveis. Não assumir que ofuscação atende à LGPD ou protege contra acessos privilegiados.
- **SLOs e observabilidade:** definir metas mensuráveis de latência, disponibilidade, detecção, alertas, retenção e resposta.
- **Estrutura do repositório:** confirmar se a árvore abaixo é proposta de organização ou estrutura existente antes de usá-la como destino obrigatório de arquivos.

## ESTRUTURA PROPOSTA DE DIRETÓRIOS DO REPOSITÓRIO (REPOSITORY LAYOUT)
Descrição: Estrutura física proposta para os arquivos e diretórios do ecossistema SDS. Confirmar sua correspondência com o repositório real antes de usá-la como orientação de localização.
VALOR PARA A IA:
- Apresentar uma proposta de organização por responsabilidade para futura validação.
- Evitar que a IA trate caminhos ainda não confirmados como existentes ou obrigatórios.

O texto para este tópico é:

A estrutura abaixo é uma proposta de organização modular:

```text
sds-framework/
├── .ai/                                    # Governança de IA e diretrizes de desenvolvimento
│   ├── 00 - CONTEXTO.md                    # Visão estratégica e governança de IA
│   └── 01 - ARQUITETURA_DO_SISTEMA.md      # Arquitetura técnica do framework
├── database/                               # Núcleo de Banco de Dados (Oracle Database 26ai)
│   ├── migration/                          # Scripts de criação de tabelas relacionais e constraints
│   ├── plsql/                              # Tipos de objeto, métodos e pacotes PL/SQL do motor
│   │   ├── types/                          # Especificações e corpos dos tipos de objeto
│   │   └── packages/                       # Pacotes complementares aos tipos de objeto
│   ├── triggers/                           # Triggers de banco, se aprovadas para o caso de uso
│   └── security/                           # Controles de segurança, conforme estratégia aprovada
├── frontend/                               # Motor de Interface Web (Python / Django - SDUI)
│   ├── core/                               # Configurações globais do Django e middleware de Correlation ID
│   ├── sdui/                               # Interpretador de Server-Driven UI e contratos relacionais
│   ├── cache/                              # Integração com Redis, gestão de chaves e fallbacks assíncronos
│   └── services/                           # Integrações com banco ou APIs, conforme canal aprovado
├── workers/                                # Barramento de Tarefas Assíncronas (Python + Celery)
│   ├── tasks/                              # Tarefas de background (ingestão de arquivos, scrapers, lotes)
│   ├── pipeline/                           # Orquestração de dados e auto-cura (repopulação de cache)
│   └── broker/                             # Configuração de filas e integração com Redis
├── api/                                    # Camada de Exposição de Endpoints para Clientes Finais
│   └── ords/                               # Mapeamentos e scripts de configuração do Oracle REST Data Services
└── tests/                                  # Testes unitários, de integração e de carga a definir
    ├── backend/                            # Testes de rotas SDUI e Django
    ├── database/                           # Testes unitários de tipos, métodos e pacotes PL/SQL
    └── resilience/                         # Testes de resiliência e recuperação, conforme requisitos definidos
```
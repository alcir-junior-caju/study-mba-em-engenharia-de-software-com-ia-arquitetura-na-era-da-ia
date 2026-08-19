<img alt="Infografico" src="infografico.png" style="margin: 15px 0" />

# AI Gateways: Do Desenvolvimento Inicial à Governança Arquitetural

Este briefing sintetiza a evolução do uso de Inteligência Artificial (IA) em sistemas de software, partindo da integração simples via SDKs nativos até a implementação de camadas sofisticadas de controle conhecidas como AI Gateways. O documento explora os desafios da escala, a necessidade de padronização e as ferramentas que possibilitam uma arquitetura resiliente e governável.

## Sumário Executivo

A integração de IA geralmente começa de forma simples, com chamadas diretas aos SDKs dos provedores. No entanto, conforme o uso cresce, surge o "espalhamento da decisão técnica", onde detalhes de infraestrutura (tokens, payloads, retries) contaminam a regra de negócio. A solução para essa fragilidade é o **AI Gateway**, uma fronteira arquitetural que centraliza e padroniza o acesso aos modelos.

A transição para um gateway permite que a aplicação pare de solicitar modelos físicos (ex: GPT-4) e passe a pedir capacidades (ex: "analista financeiro"). Ferramentas como o **LiteLLM** facilitam essa jornada, oferecendo compatibilidade entre múltiplos provedores e permitindo a implementação de políticas de governança, controle de custos e resiliência técnica (fallback). O sucesso dessa arquitetura depende da equivalência entre modelos em cenários de contingência, garantindo que a qualidade da resposta se mantenha consistente mesmo em caso de falha do provedor principal.

---

## 1. O Desafio da Escala e o Problema do Acoplamento

No início do desenvolvimento (PoCs e protótipos), o uso de bibliotecas nativas de fabricantes (OpenAI, Google Gemini, Anthropic) é produtivo. Contudo, a expansão do uso de IA revela problemas estruturais graves:

- **Espalhamento da Decisão Técnica:** Detalhes como controle de temperatura, esquemas de dados, limites de conexão e contagem de tokens deixam de ser isolados e passam a estar espalhados por toda a arquitetura.
- **Incompatibilidade de Interfaces:** SDKs diferentes possuem assinaturas e parâmetros distintos (ex: `max_tokens` vs. `max_new_tokens`), tornando a troca de provedores uma tarefa complexa que exige alterações em múltiplos pontos do sistema.
- **Fragilidade Operacional:** Sem uma camada centralizada, é difícil aplicar regras globais de segurança, rastrear gastos financeiros por módulo ou implementar rotas de contingência de forma eficiente.

---

## 2. Definindo o AI Gateway

O AI Gateway não é apenas um "helper" ou uma função de encapsulamento; é uma **fronteira arquitetural**. Sua função é centralizar, padronizar e controlar o acesso aos motores cognitivos.

### Mudança de Paradigma: De Modelos para Capacidades

A principal evolução proporcionada pelo gateway é a mudança no vocabulário do sistema. A aplicação deixa de conhecer o modelo físico e passa a solicitar uma capacidade interna:

| Abordagem Tradicional (Modelo Físico) | Abordagem com Gateway (Capacidade) |
| :------------------------------------ | :--------------------------------- |
| Chamar `gpt-4o`                       | Chamar `tradutor_rapido`           |
| Chamar `claude-3-opus`                | Chamar `analista_financeiro`       |
| Chamar `gemini-1.5-flash`             | Chamar `sumarizador_contratos`     |

Essa abstração permite que a equipe de arquitetura troque o modelo por trás da capacidade (por questões de custo ou performance) sem que a aplicação precise ser alterada ou sequer saiba da mudança.

---

## 3. Níveis de Maturidade e a Ferramenta LiteLLM

O documento identifica uma progressão na maturidade arquitetural, utilizando o LiteLLM como peça central para ilustrar essa evolução.

### 3.1. LiteLLM como SDK (Compatibilidade Local)

Funciona como uma biblioteca dentro da própria aplicação.

- **Objetivo:** Reduzir o acoplamento com SDKs específicos.
- **Funcionamento:** Oferece uma interface única para múltiplos provedores, traduzindo a requisição internamente.
- **Benefício:** Facilita a troca de modelos no código com uma única assinatura de função (`completion`).

### 3.2. LiteLLM como Proxy (Serviço Centralizado)

O gateway passa a rodar como um serviço separado (via Docker, por exemplo), acessado via HTTP.

- **Topologia:** `Aplicação -> Proxy -> Provedor`.
- **Autenticação:** Introduz o conceito de Virtual Keys. A aplicação usa uma chave gerada pelo gateway, que por sua vez mascara e protege as chaves reais dos provedores (Anthropic, OpenAI, etc.).
- **Centralização:** Permite que múltiplos sistemas (Gestão de Frota, CRM, Chatbots) consumam IA através de um único ponto de controle.

---

## 4. Fluxo de Requisição e Governança

Dentro de um AI Gateway maduro, uma requisição passa por diversas etapas de validação e processamento:

1. **Validação de Chave Virtual:** Verifica se a chave existe, está ativa e pertence a qual departamento ou aplicação.
2. **Aplicação de Budgets e Rate Limits:** Controla o orçamento disponível e o limite de chamadas (por segundo ou por tokens) para evitar que um módulo consuma todos os recursos da empresa.
3. **Roteamento (Router):** O componente central decide para onde enviar a chamada. Ele mapeia o _Model Group_ (nome lógico/capacidade) para o _Deployment_ (configuração concreta do modelo no provedor).
4. **Normalização e Registro:** O gateway recebe a resposta, traduz para um formato padrão, registra custos e latência de forma assíncrona e devolve o resultado à aplicação.

---

## 5. Resiliência e o Conceito de Fallback

A resiliência em IA exige mecanismos técnicos que protejam o sistema contra falhas dos provedores:

- **Timeout:** Decisão de quanto tempo esperar por uma resposta, variando conforme o caso de uso (ex: dashboards rápidos vs. processamento de lotes noturnos).
- **Retry:** Tentativas automáticas em caso de falhas temporárias de conexão ou picos de acesso, devendo ser usado com critério para não sobrecarregar o provedor.
- **Fallback Técnico:** Utilização de um modelo de backup quando o principal falha.

### O Problema da Equivalência

O fallback apresenta um desafio crítico: **modelos diferentes não são idênticos.**

- **Falha de Comportamento:** Um modelo reserva pode não seguir instruções estritas (ex: responder em XML puro) que o modelo principal seguia, quebrando o processamento da aplicação.
- **Trade-offs:** A escolha do modelo de backup deve considerar se a nova resposta é "boa o suficiente" para o caso de uso, equilibrando custo, latência e precisão.

> "Não adianta ter um fallback apenas e fazer com que esse fallback chame um modelo que não vai resolver o problema... a redundância de modelos tem que ser equivalente."

---

## Conclusão: A Evolução da Pergunta Arquitetural

Com a introdução dos AI Gateways, a pergunta fundamental da arquitetura de sistemas evolui:

- **Antes:** _"Como eu chamo este modelo de IA?"_
- **Agora:** _"Como a minha arquitetura controla, padroniza, protege e evolui o uso de IA dentro do sistema?"_

O uso de gateways e proxies (como LiteLLM, Kong AI Gateway ou AWS API Gateway) transforma a IA de uma funcionalidade isolada em uma capacidade compartilhada, governável e resiliente, essencial para sistemas em nível de produção.

### [Assista ao resumo em vídeo](https://github.com/user-attachments/assets/108fcd1d-bcd4-49a0-a9f7-be615de0c81e)

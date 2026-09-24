# Guia de Architecture Decision Record (ADR)

Um **Architecture Decision Record (ADR)** é um documento curto em texto simples (geralmente Markdown) que captura uma decisão de arquitetura relevante tomada em um projeto de software, juntamente com o seu contexto e suas consequências.

---

## 1. Template em Branco (Copiar e Colar)

Salve os arquivos com o padrão de nomenclatura sequencial:

`docs/adr/NNNN-titulo-em-kebab-case.md` (Ex.: `docs/adr/0004-escolha-do-banco-vetorial.md`).

```markdown
# [Número Sequencial]. [Título Claro da Decisão em Modo Imperativo]

* **Status:** [Proposto | Aceito | Rejeitado | Depreciado | Substituído por ADR-XXXX]
* **Data:** AAAA-MM-DD
* **Autores:** [Nome(s) de quem elaborou]
* **Decisores:** [Time, Tech Lead, Engenheiros envolvidos na aprovação]
* **Contexto Técnico / Requisitos:** [PRs, Issues, RFCs relacionados]

---

## 1. Contexto e Declaração do Problema
[Descreva o problema de negócio ou desafio técnico que motivou a decisão. Explique quais forças e restrições estavam em jogo: custo, prazo, escalabilidade, latência, curva de aprendizado da equipe, etc. Evite justificar a solução aqui; foque na dor.]

## 2. Opções Consideradas
* **Opção 1: [Nome da alternativa]**
  * *Prós:* [Vantagens]
  * *Contras:* [Desvantagens / Limitações]
* **Opção 2: [Nome da alternativa]**
  * *Prós:* [Vantagens]
  * *Contras:* [Desvantagens / Limitações]
* **Opção 3: [Nome da alternativa]**
  * *Prós:* [Vantagens]
  * *Contras:* [Desvantagens / Limitações]

## 3. Decisão Escolhida
[Descreva claramente a opção escolhida e a justificativa principal da escolha diante das restrições apresentadas.]

## 4. Consequências e Trade-offs

### Impactos Positivos (O que ganhamos)
* [Benefício direto 1]
* [Benefício direto 2]

### Impactos Negativos e Riscos (O que assumimos)
* [Custo operacional, complexidade ou débito técnico consciente]
* [Mitigação planejada para cada risco levantado]

### Itens de Ação / Próximos Passos
- [ ] [Tarefa 1 de implementação ou infraestrutura]
- [ ] [Tarefa 2 de validação ou documentação complementar]
```

---

## 2. Exemplo Real Preenchido para Aula

Abaixo está uma ADR realista de um sistema de comércio eletrônico migrando de consultas síncronas para mensageria assíncrona.

```markdown
# 0003. Adoção de Mensageria com RabbitMQ para Notificação de Pedidos

* **Status:** Aceito
* **Data:** 2026-09-24
* **Autores:** Ana Silva (Backend Engineer)
* **Decisores:** Carlos Mendes (Tech Lead), Time de Checkout & Notificações
* **Contexto Técnico:** Issue #142 (Timeout no fechamento de carrinho em picos de tráfego)

---

## 1. Contexto e Declaração do Problema

Atualmente, ao concluir uma compra no endpoint `POST /api/v1/orders`, a aplicação executa chamadas HTTP síncronas em sequência para três serviços externos:
1. Emissão de Nota Fiscal (`Fiscal-Service`);
2. Notificação por E-mail/WhatsApp (`Notification-Gateway`);
3. Baixa em Sistema Legado de Estoque (`ERP-Adapter`).

Durante as campanhas promocionais, o tempo médio de resposta do checkout saltou de 320 ms para 4,8 segundos, gerando timeouts de cliente (HTTP 504) e desistência de compra. Além disso, se o `Notification-Gateway` estiver fora do ar, o pedido inteiro falha, gerando inconsistência com a operadora de pagamentos.

Precisamos desacoplar as rotinas secundárias do fluxo crítico de compra, garantindo resposta em menos de 500 ms com garantia de entrega eventual das notificações.

## 2. Opções Consideradas

* **Opção 1: Manter HTTP com chamadas assíncronas via Threads em memória**
  * *Prós:* Não adiciona infraestrutura externa; implementação imediata.
  * *Contras:* Se o pod da API reiniciar ou sofrer crash durante o deploy, os eventos em memória são perdidos irremediavelmente; sem garantia de retentativa (retry) ou dead-letter.

* **Opção 2: RabbitMQ (AMQP broker)**
  * *Prós:* Baixa latência; modelo maduro de exchanges/filas com Dead Letter Queues (DLQ); consumo leve de memória; excelente suporte no ecossistema de bibliotecas do time.
  * *Contras:* Requer gerenciamento de infraestrutura (broker gerenciado ou cluster K8s); retenção de mensagens não é permanente (ao contrário de logs distribuídos).

* **Opção 3: Apache Kafka**
  * *Prós:* Alta vazão com particionamento; replay histórico de eventos; persistência em disco de longo prazo.
  * *Contras:* Alta complexidade operacional e de configuração inicial para a escala atual da aplicação (menos de 500 mensagens/segundo); curva de aprendizado íngreme para o time júnior/pleno.

## 3. Decisão Escolhida

Decidimos adotar o **RabbitMQ** como message broker para o processamento assíncrono pós-pagamento.

A escolha decorre do equilíbrio entre simplicidade operacional e confiabilidade: nossa volumetria estimada para os próximos 18 meses não justifica o overhead do Apache Kafka, e o uso de threads em memória apresenta risco inaceitável de perda de dados transacionais.

## 4. Consequências e Trade-offs

### Impactos Positivos
* O endpoint de criação de pedidos passa a apenas persistir o pedido em status `CONFIRMED`, publicar o evento `OrderPlacedEvent` no RabbitMQ e retornar HTTP 201 em ~150 ms.
* Falhas temporárias no gateway de e-mail ou ERP não impactam mais a experiência de compra do cliente final.
* Facilidade de adicionar novos consumidores (ex.: módulo de auditoria antifraude) sem alterar o código do serviço de checkout.

### Impactos Negativos e Riscos
* **Consistência Eventual:** O cliente não recebe mais a confirmação instantânea de emissão de NF na tela de sucesso. A UI precisará exibir status de "Processando envio".
* **Complexidade Operacional:** É necessário configurar observabilidade (métricas de fila acumulada, alarmes de consumer lag) e monitorar a Dead Letter Queue para evitar perda silenciosa de eventos.

### Itens de Ação
- [ ] Provisionar instância de RabbitMQ no cluster de homologação via Helm Chart.
- [ ] Implementar política de retentativa exponencial com 3 tentativas antes de enviar para DLQ (`orders.dead-letter`).
- [ ] Criar dashboard no Grafana monitorando `rabbitmq_queue_messages_ready` e tempo de consumo.
```

---

## 3. Automação via CLI (`adr-tools`)

O utilitário de terminal `adr-tools` padroniza a criação sequencial, vinculação e substituição de decisões sem conflitos manuais de nomenclatura.

### Instalação
```bash
# macOS (Homebrew)
brew install adr-tools

# Linux (Debian/Ubuntu via apt ou clonagem direta)
sudo apt-get install adr-tools
```

### Comandos Essenciais

1. **Inicializar o repositório de ADRs:**
   ```bash
   # Cria o diretório docs/adr e gera o primeiro registro (0001-record-architecture-decisions.md)
   adr init docs/adr
   ```

2. **Criar uma nova decisão:**
   ```bash
   # Gera automaticamente o arquivo sequencial (ex: 0004-usar-redis-para-cache.md)
   adr new Usar Redis para Cache de Sessao
   ```

3. **Substituir ou depreciar uma decisão anterior:**
   ```bash
   # Substitui a decisão 0002 pela nova decisão 0005 e conecta ambas mutuamente
   adr new -s 2 Migrar de PostgreSQL para MongoDB
   ```

4. **Gerar índice visual em Markdown:**
   ```bash
   # Atualiza automaticamente o README.md da pasta docs/adr com a lista de todas as decisões e status
   adr generate toc > docs/adr/README.md
   ```

---

## 4. Validação Contínua com GitHub Actions

Para garantir que novos registros sigam os padrões de nomenclatura e seções obrigatórias, utilize o workflow abaixo em `.github/workflows/validate-adr.yml`:

```yaml
name: Validar Padronizacao de ADRs

on:
  pull_request:
    paths:
      - 'docs/adr/**.md'

jobs:
  lint-adrs:
    name: Validar Nomenclatura e Metadados
    runs-on: ubuntu-latest
    steps:
      - name: Checkout do Repositório
        uses: actions/checkout@v4

      - name: Verificar Convencao de Nomes e Secoes Obrigatorias
        shell: bash
        run: |
          echo "Validando arquivos em docs/adr/..."
          STATUS=0

          for file in docs/adr/[0-9]*.md; do
            [ -f "$file" ] || continue
            echo "Checando: $file"

            # 1. Valida nome do arquivo (ex: 0001-nome-kebab.md)
            if ! [[ $(basename "$file") =~ ^[0-9]{4}-[a-z0-9-]+\.md$ ]]; then
              echo "::error file=$file::Nome de arquivo invalido. Use o padrao 0001-titulo-kebab-case.md"
              STATUS=1
            fi

            # 2. Valida presenca dos campos criticos
            for required_section in "## 1. Contexto" "## 2. Opções Consideradas" "## 3. Decisão Escolhida" "## 4. Consequências"; do
              if ! grep -q "$required_section" "$file"; then
                echo "::error file=$file::Secao obrigatoria ausente: '$required_section'"
                STATUS=1
              fi
            done

            # 3. Valida se o status esta declarado
            if ! grep -Eq "\* \*\*Status:\*\* (Proposto|Aceito|Rejeitado|Depreciado|Substituído)" "$file"; then
              echo "::error file=$file::Campo 'Status' ausente ou com valor nao padronizado."
              STATUS=1
            fi
          done

          exit $STATUS
```
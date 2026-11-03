# Plano: deploy da plataforma FCG na AWS

## Objetivo

Preparar e publicar os cinco projetos da plataforma FIAP Cloud Games na AWS, com EKS como ambiente Kubernetes, GitHub Actions para CI/CD e serviços de dados/mensageria temporariamente executados em pods. Usar recursos gerenciados da AWS para os demais componentes quando fizer sentido e atender aos requisitos da entrega.

## Decisões confirmadas

- **Compute/orquestração:** Amazon EKS.
- **Provisionamento inicial:** AWS Console.
- **CI/CD:** GitHub Actions nos repositórios.
- **Imagens de containers:** Amazon ECR.
- **Dados e mensageria, nesta fase:** SQL Server, MongoDB e RabbitMQ em pods no EKS.
- **Recursos AWS adicionais:** usar serviços gerenciados quando adequados aos requisitos, por exemplo OpenSearch Service ou ElastiCache, após confirmar necessidade e compatibilidade.
- **Escopo:** FCGCatalog, FCGUser, FCGPayment, FCGNotification e FCGInfra.

## Auditoria inicial

- Não foram encontradas referências a LocalStack nos repositórios examinados.
- As aplicações já usam SQL Server, MongoDB e RabbitMQ; não há migração para serviços de mensagens AWS prevista nesta fase.
- Os manifests atuais do Kubernetes usam nomes de imagem locais e precisam apontar para imagens publicadas no ECR.
- SQL Server e RabbitMQ têm manifests Kubernetes existentes, mas precisam de armazenamento persistente e configuração apropriada para EKS.
- O FCGCatalog usa MongoDB, mas não foi encontrado um workload/manifests de MongoDB na infraestrutura atual.
- As configurações de conexão, credenciais, probes, portas e dependências entre serviços precisam ser revistas para o ambiente AWS.

## Passos de implementação

1. **Consolidar requisitos e auditar os projetos**
   - Conferir critérios finais da Fase 3, entregáveis e estado dos cinco repositórios.
   - Confirmar se OpenSearch/fuzzy search e Redis são requisitos obrigatórios e quais comportamentos de busca e consistência devem ser demonstrados.
   - Identificar portas, variáveis de configuração, health checks, dependências e testes existentes por serviço.

2. **Preparar a conta e o ambiente AWS**
   - Confirmar conta, região, permissões, orçamento e limites de serviço.
   - Criar a rede e o cluster EKS pelo AWS Console, incluindo node group e acesso seguro ao cluster.
   - Registrar as escolhas e configurações necessárias para repetir o setup.

3. **Criar ECR e identidade de CI/CD**
   - Criar repositórios ECR para as imagens das APIs e dependências customizadas, se aplicável.
   - Configurar GitHub Actions com federação OIDC para assumir uma IAM Role, sem armazenar chaves AWS permanentes nos repositórios.
   - Restringir permissões da role aos repositórios, branches/ambientes e ações de ECR/EKS que o pipeline precisa.

4. **Adaptar workflows GitHub Actions**
   - Nos repositórios das APIs, executar restore, build e testes; construir e publicar imagens versionadas no ECR.
   - Preferir SHA do commit ou tags imutáveis para deploy, não depender de `latest`.
   - Configurar o deploy no EKS após a validação do build/testes e definir um caminho de rollback.

5. **Preparar Kubernetes para EKS**
   - Ajustar manifests em FCGInfra para imagens do ECR, namespaces, configurações, probes, recursos e serviços.
   - Corrigir as conexões entre APIs, SQL Server, MongoDB e RabbitMQ para os nomes DNS internos do cluster.
   - Configurar Secrets sem credenciais versionadas; avaliar AWS Secrets Manager/External Secrets conforme compatibilidade e escopo.
   - Adicionar MongoDB e persistência para SQL Server, MongoDB e RabbitMQ usando PersistentVolumes/Claims e a classe de armazenamento apropriada.
   - Planejar backups e ensaiar restauração. Documentar os riscos operacionais dos bancos em pods e a natureza temporária dessa opção.

6. **Fazer o primeiro deploy e validar os fluxos**
   - Publicar imagens e aplicar os manifests pelo pipeline.
   - Validar readiness/liveness, logs, conectividade interna, acesso externo e reinício/atualização dos pods.
   - Testar cadastro/autenticação, consulta ao catálogo, compra, processamento de pagamento e notificações.
   - Corrigir inconsistências entre configurações e contratos de eventos antes da etapa de busca avançada.

7. **Implementar e validar busca avançada**
   - Confirmar requisitos de indexação e escolher OpenSearch Service ou outra opção compatível.
   - Implementar o endpoint, fuzzy search e relevância conforme critérios da entrega.
   - Definir e testar sincronização/consistência do catálogo com o índice.
   - Adicionar Redis/ElastiCache apenas se exigido ou justificado pelos requisitos; testar invalidação e consistência de cache.

8. **Testes integrados e revisão**
   - Testar fluxos ponta a ponta, consistência do índice/cache, persistência após reinícios e deploy/rollback.
   - Validar permissões IAM/GitHub, exposição de endpoints, uso de Secrets, logs e backups/restauração.
   - Corrigir problemas encontrados e registrar limitações conhecidas.

9. **Documentação e demonstração**
   - Documentar arquitetura, setup AWS, repositórios ECR, workflows, variáveis/secrets e procedimentos de deploy/rollback.
   - Preparar roteiro de Live Deploy e demonstrar os fluxos exigidos, busca e comportamento dos pods.
   - Revisar os entregáveis finais e o vídeo.

## Cronograma de referência e situação

O cronograma fornecido foi escrito para 09/10–17/11. Considerando a data atual do workspace (03/11/2026), a etapa inicial e a janela de CI/CD já passaram, e a janela de busca está em andamento. Replanejar a execução restante com a equipe; não considerar etapas anteriores concluídas sem validação.

| Etapa original | Período | Foco | Situação em 03/11 |
|---|---|---|---|
| Consolidar conteúdo | 09–16/10 | Aulas, requisitos e auditoria da Fase 3 | Período encerrado; confirmar itens pendentes |
| CI/CD e infraestrutura | 17–26/10 | Pipelines, ECR, Secrets e deploy automatizado | Período encerrado; priorizar/validar imediatamente |
| Busca avançada | 27/10–05/11 | OpenSearch, endpoint, fuzzy search e relevância | Em andamento; confirmar requisito e status |
| Testes integrados | 06–11/11 | Redis, consistência, segurança e atualização de pods | Próxima etapa; manter janela curta |
| Documentação e vídeo | 12–17/11 | Live Deploy e entregáveis | Janela final |

## Considerações e critérios de conclusão

- A infraestrutura AWS pode gerar custos; confirmar orçamento e desligar recursos não usados.
- Manter bancos em pods é uma decisão temporária e requer armazenamento persistente, backups testados e monitoramento.
- O deploy só está pronto quando os workflows publicam imagens no ECR e fazem rollout no EKS sem credenciais AWS estáticas.
- As quatro APIs precisam iniciar, comunicar-se com as dependências e completar os fluxos essenciais.
- Dados devem sobreviver a reinícios dos pods; uma restauração de backup deve ser demonstrada.
- Busca avançada e Redis entram no escopo de implementação conforme os requisitos finais, não por suposição.
- Documentação deve permitir repetir o deploy e explicar limitações e recuperação.

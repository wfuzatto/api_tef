# Estudo técnico — Getnet Payment Link x E-commerce e integração Auttar TEF

**Data da pesquisa:** 06/11/2026  
**Aplicação proposta:** hotelaria, AcquaVale (venda online de ingressos e liberação de acesso), balcões de atendimento/PDVs e serviços associados.  
**Natureza:** estudo de arquitetura e contratação; não constitui homologação nem substitui contrato e documentação técnica vigente de cada credencial.

## 1. Resumo executivo e veredito

A expressão "e-commerce da Getnet" não identifica uma API única. Há pelo menos três caminhos técnicos distintos:

1. **Link de pagamento / Get Pay / Payment Link API**: um link público conduz o comprador a uma página de pagamento da Getnet. Pode ser criado manualmente no portal/app ou programaticamente quando a conta dispõe da Payment Link API. O fluxo é útil para cobranças avulsas, vendas remotas, recuperação de carrinho e pagamentos assistidos.
2. **Web Checkout/Get Checkout**: o e-commerce cria uma intenção de pagamento, recebe identificadores e incorpora ou redireciona o cliente para o checkout seguro operado pela Getnet (redirect, iframe ou lightbox, conforme produto contratado e documentação aplicável). É uma opção naturalmente integrada à jornada do website.
3. **API direta / Global API (SEP)**: o integrador controla mais do fluxo de transação e operações subsequentes, mas assume maior custo de engenharia, requisitos de segurança/PCI e testes.
4. **Auttar TEF**: solução voltada ao pagamento PRESENCIAL com terminal/pinpad e uma automação comercial/PDV. Não equivale a um link e nem é substituta da API de e-commerce. Para uma integração omnicanal, deve alimentar o mesmo cadastro de pedidos/financeiro via um adaptador próprio.

**Conclusão sobre boleto:** é tecnicamente suportado no Brasil pela Payment Link API atual, que prevê payment.bankslip, e a página comercial Getnet também informa boleto no Link de Pagamento. Portanto, não é correto afirmar que "boleto só existe no checkout de e-commerce". Entretanto, **não está comprovado que boleto está liberado no CNPJ/contrato/API/portal específicos**: a habilitação pode variar por produto, convênio, banco, preço e credenciamento. Há material comercial de 2025 condicionando boleto e Pix a convênio Santander. Validar com a equipe Getnet antes de considerar a funcionalidade disponível em produção. [G1][G2][G3][G12]

**Recomendação para AcquaVale:** Web Checkout integrado ao pedido para compras no próprio site; Payment Link como canal auxiliar de cobrança remota; Auttar TEF para caixa presencial, com confirmação e conciliação no backend. A autorização de acesso deve depender de confirmação verificável do pagamento, jamais somente de um redirecionamento ou da emissão de boleto.

## 2. Mapa das soluções e diferenças

| Critério | Payment Link (API/portal) | Web Checkout de e-commerce | Global API direta | Auttar TEF |
| --- | --- | --- | --- | --- |
| Canal | Remoto/online | Site e aplicativo web | Site/backend, inclusive caminhos omnichannel | Presencial: PDV/pinpad |
| Experiência | Checkout em URL compartilhável | Redirect, iframe ou lightbox Getnet | Interface sob maior responsabilidade do integrador | Interface local e fluxo do terminal/pinpad |
| Integração | Manual no portal ou POST de criação via API | Backend cria payment intent + front integra checkout | Backend chama APIs de pagamento; tokenização/PCI conforme desenho | PDV conversa com CTFClient por interface contratada |
| Meios de pagamento BR documentados | Crédito, débito, Pix, boleto; wallets conforme produto/país | Crédito, débito, Pix, boleto e variações segundo contrato | Depende do fluxo/API e habilitação | Cartões/voucher e modalidades contratadas; Pix e outros exigem verificação específica |
| Boleto | **Sim, documentado**, depende de habilitação | **Sim, documentado**, depende de habilitação | Consultar API específica e adquirência contratada | Não presumir boleto disponível no pinpad |
| Pix | Sim, documentado | Sim, documentado | Sim, dependendo de fluxo | Exige confirmação de capacidade do pacote Auttar, pinpad e adquirência |
| Parcelamento | Configuração por bandeira e esquema | Configuração via merchant/checkout | Controle conforme produto | Operações crédito à vista, parcelado lojista/emissor conforme habilitação |
| Personalização | Restrita às opções do link, produto e checkout operado pelo provedor | Mais integrada ao visual, mas dados sensíveis ficam com Getnet | Máxima flexibilidade e maior responsabilidade PCI | Identidade do PDV e interface TEF conforme SDK/protocolo |
| Segurança | Getnet hospeda entrada sensível | Reduz escopo de dados sensíveis no estabelecimento | Requisitos PCI/3DS/antifraude e responsabilidade técnica maiores | CTFClient e hardware homologado; não armazenar dados sensíveis |
| Acompanhamento | Ordens do link, consultas; contrato pode oferecer avisos | Webhook de pagamento e consulta por pagamento | Webhook e API de consulta | Retorno CTFClient, NSU, autorização, confirmação e conciliação |
| Automação principal | Cobrança ad hoc por URL | Checkout nativo da loja | Operações financeiras avançadas | Vendas presenciais com pinpad |
| Melhor uso no projeto | Cobranças assistidas por recepção/WhatsApp | Jornada principal de ingressos | Fluxos especiais e futuro hub financeiro | Bilheteria, recepção e caixa |

**Importante sobre nomenclatura:** "Get Pay"/"Link de Pagamento" comercial e "Payment Link API" do portal global não devem ser tratados como contratos, credenciais e APIs automaticamente intercambiáveis. A oferta brasileira pode evoluir antes/depois da documentação global. Confirmar qual produto está provisionado.

## 3. Getnet Payment Link — análise funcional e técnica

### 3.1 Jornada

Sistema cria um pedido -> backend autenticado cria link -> salva link_id/short_id e associação com pedido -> envia URL -> cliente escolhe método de pagamento -> Getnet processa -> sistema verifica resultado de forma confiável -> aprova pedido -> entrega ingresso/QR ou confirma serviço.

O link pode conter catálogo e valor, configuração de meios, data de expiração e limite de pedidos. Um link "custom" exige produtos; uma criação retorna identificadores link_id e short_id. A documentação cita estados de pedido como pending, paid, approved, refunded e denied. Esses estados devem ser normalizados na aplicação e não interpretados por aproximação. [G1][G4]

### 3.2 API e autenticação

- Fluxo OAuth 2.0 client_credentials, usando client_id e client_secret.
- Autenticação documentada: POST https://api.globalgetnet.com/authentication/oauth2/access_token com grant_type=client_credentials (application/x-www-form-urlencoded). Token com validade nominal de cerca de 60 minutos. Nunca chamar do JavaScript do navegador com o secret. [G5]
- Criação de link: POST /payment-links; os exemplos atuais mostram a URL base e prefixos específicos do produto (por exemplo, /dpy/payment-link/v1). Obter o BASE URL de sandbox e produção diretamente do Swagger da versão contratada.
- Há uma **divergência de documentação**: a visão geral diz que dados do seller/tenant vêm do token, enquanto um exemplo de criação inclui headers x-seller-id, country e tenant. **Não remover nem inventar headers em produção**: testar contra o OpenAPI/tenant efetivamente habilitado, com suporte Getnet. [G1][G4]
- Os valores dos itens são inteiros em centavos, currency BRL.
- Configurações de pagamento incluem payment.credit, payment.debit, payment.bankslip e payment.instant_payment. Crédito exige configuração de bandeiras no formato documentado. [G4][G6]

**Exemplo ilustrativo do corpo (validar campos e headers na versão contratada):**

~~~json
{
  "label": "ACQUA20261106A",
  "type": "custom",
  "max_orders": 1,
  "request_delivery_address": false,
  "products": [
    {
      "product_type": "service",
      "title": "Ingresso AcquaVale",
      "description": "Ingresso de parque aquático",
      "order_prefix": "ACQUA",
      "amount": 15000,
      "quantity": 1
    }
  ],
  "payment": {
    "bankslip": {"enabled": true},
    "instant_payment": {"enabled": true}
  },
  "currency": "BRL"
}
~~~

O exemplo demonstra que **um link pode ser configurado somente com boleto e Pix**, mas não garante aprovação de requisição em uma conta sem os serviços liberados. Não inventar a URL final com base apenas no short_id: usar a URL pública/documentação efetivamente retornada/construída conforme versão do checkout.

### 3.3 Consulta, expiração e conciliação

A API documenta criar, listar, consultar e atualizar links, recuperar um link por short_id, listar/consultar pedidos associados e obter recibos. Há configurações reutilizáveis de negócio e operações para imagens. O atributo max_orders não substitui uma trava transacional local: para um pedido do parque, usar max_orders=1 e, adicionalmente, travar a liberação idempotente do pedido. [G1][G4]

**Ponto não comprovado:** não assumir que todo evento de Payment Link chega por um webhook próprio equivalente ao Web Checkout/Global API. Há uma documentação geral de Webhooks da Getnet, mas é necessário verificar se as transações de link do contrato em questão são emitidas nessa inscrição. Até homologar, implementar consulta de pedidos/estado como fallback, respeitando limites e atrasos de processamento. [G1][G8]

### 3.4 Boleto e Pix: interpretação correta

- Criar boleto **não significa receber**. Manter pedido como aguardando pagamento até evento/consulta positiva e correlacionada.
- Pix pode liquidar rapidamente, mas callback/consulta pode atrasar; não usar apenas a tela de sucesso do comprador.
- Vencimento, expiração, pagamento pós-vencimento, juros, protesto, reemissão, estorno e tratamento bancário devem ser contratados/testados. A documentação pública verificada não autoriza generalizar todos esses recursos para Payment Link.
- O pacote Getnet de 2025 analisado registra tarifação de boletos por **emissão** e condição de convênio Santander para boleto e Pix. Isso é evidência de uma oferta específica, **não** regra universal de 2026. [G12]

### 3.5 Cancelamento e estorno

A Getnet possui guia específico para cancelamentos/estornos de pagamentos por link. Em transações com cartão, o cancelamento no mesmo dia (D+0), antes do fechamento, é integral; em D+1 ou posterior pode haver estorno parcial, sujeito a país, bandeira, método e disponibilidade. Para métodos não cartão não presumir o mesmo endpoint/semântica. Antes de implementar estorno, persistir payment_id, payment_method, identificadores de origem e estado da operação. [G13]

## 4. Web Checkout (e-commerce) — arquitetura e comportamento

### 4.1 Modelos

- **Redirect:** o backend cria a intenção; o cliente abre redirect_url para página hospedada Getnet.
- **Iframe:** o checkout seguro aparece dentro da página da loja.
- **Lightbox:** um componente/modal de checkout é carregado sobre a loja.
- O produto comercial brasileiro também faz referência a opções iFrame e API/checkout transparente; **não confundir marketing do pacote com um único SDK universal**. [G7][G10]

### 4.2 Integração documentada

1. Obter credenciais próprias de sandbox e produção, provisionadas para o produto correto.
2. Obter token OAuth 2.0 no backend.
3. Criar a intenção de pagamento por POST /payment-intent com order_id e total calculado no servidor (e demais dados exigidos).
4. Receber payment_intent_id e redirect_url; integrar frontend conforme formato escolhido.
5. O comprador finaliza pagamento na interface Getnet.
6. Webhook de autorização/negação notifica o backend; correlacionar por order_id e payment_intent_id.
7. Quando autorizado, obter payment_id e, se necessário, usar Global API para consulta, captura, cancelamento/estorno.

O guia da Getnet mostra o loader em https://www.globalgetnet.com/digital-checkout/loader.js (produção) ou host de homologação e modos iframe/lightbox. A integração deverá usar URLs e instruções da documentação da conta específica; não copiar literalmente exemplo de outro país. [G7]

### 4.3 Webhooks do Web Checkout

A configuração técnica permite callback HTTPS e autenticação Basic para o webhook do Web Checkout. A documentação manda responder **HTTP 200**. Em falha de recebimento, descreve cinco novas tentativas em intervalos progressivos e depois até 72 horas em ciclos horários. Não usar a página success_url como prova de recebimento. [G9]

**Atenção: dois contratos de webhook distintos:**
- **Web Checkout:** documentação própria recomenda HTTP 200 ao receber.
- **Global API/Webhook Management:** documentação própria especifica HTTP 204 e oferece Basic, OAuth e Bearer. Eventos incluem APPROVED_TRANSACTIONS, CAPTURED_TRANSACTIONS, PENDING_TRANSACTIONS, BOLETO_PAID_TRANSACTIONS, BOLETO_UPDATED_TRANSACTIONS, PIX_UPDATED_TRANSACTIONS, REFUNDED_TRANSACTIONS, etc. [G8][G9]

Não implementar um único callback genérico sem conferir contrato, payload, autenticação, código esperado e esquema de tentativas da família de API habilitada. Guardar payload bruto saneado, chave de deduplicação e resultado da reconciliação.

### 4.4 Captura e pré-autorização

Para hotelaria, a Global API documenta operações de venda imediata e pré-autorização com captura posterior. O Web Checkout permite parametrizar pré-autorização conforme o contrato/país. A API de gerenciamento utiliza, entre outros, POST /dpm/payments-gwproxy/v2/payments/capture para captura, e POST /dpm/payments-gwproxy/v2/payments/cancel para cancelamento/estorno. Existe diferença entre autorização, captura, liquidação e disponibilização financeira; para ingresso de acesso imediato, confirmar a condição de "pago" definida pela adquirência. [G7][G11]

### 4.5 API direta: quando escolher

Considerar apenas quando o desenho exigir maior controle sobre cartão/tokenização, gestão financeira complexa, recorrência ou experiência própria. Requer revisão do escopo PCI DSS, tratamento de tokens, 3DS/antifraude, observabilidade, controle de idempotência e homologação. Não enviar PAN/CVV ao backend próprio sem arquitetura de conformidade definida. A Getnet dispõe de cofre/tokenização e recorrência como serviços, mas habilitação e disponibilidade variam por plano; a ficha comercial de 2025 indica, por exemplo, pacote sem cofre/renovação de cartão. [G10][G12]

## 5. Financeiro, tarifas e contratação

### 5.1 Evidência pública (NÃO é cotação do estabelecimento)

- Página brasileira Link de Pagamento consultada em 06/11/2026: oferta "Inicial" com indicação de R$ 1,00 por link pago; oferta "Básico" R$ 28/mês, 60 transações de cartão, R$ 0,50 excedente. Sujeito a elegibilidade e taxas da negociação. [G3]
- PDF público Getnet publicado em 2025: pacote exemplificativo R$ 28/mês, 60 operações de cartão e R$ 0,50 excedente; R$ 0,50 para Pix/QR/Wallet/Boleto, 0 inclusas; boleto cobrado por emissão; em sua redação, transações de cartão enviadas pelo site podem ser tarifadas aprovadas ou não, enquanto link de pagamento somente quando aprovado. **Não generalizar para a proposta comercial vigente ou para todas as modalidades.** [G12]
- A Getnet também comercializa condições de recebimento, antecipação e MDR/taxas de cartão, que não são equivalentes à tarifa da plataforma/gateway.

### 5.2 Itens a exigir em proposta escrita

Mensalidade, implantação, MDR crédito/débito por bandeira, parcelas lojista/emissor, custo por tentativa aprovada/negada, tarifa por link criado ou pago, tarifa por Pix, emissão e liquidação de boleto, banco emissor, antecipação, antifraude/chargeback, cofre e recorrência, consulta/webhook, estorno, conciliação, prazo de recebimento e multa/fidelidade.

Para comparar financeiramente 10 mil vendas mensais, usar fórmulas e cenários com taxas CONTRATUAIS:

- custo_link = mensalidade_link + taxa_por_link_efetivamente_tarifável + MDR + custo_pix/boleto + antecipação;
- custo_checkout = mensalidade_gateway + taxa_por_transação_elegível (inclusive negadas se aplicável) + MDR + serviços;
- custo_TEF = mensalidade/licença TEF + custo da adquirência + terminais/pinpads + suporte + comunicação;
- custo_total também precisa incorporar conversão, inadimplência de boleto, suporte humano e esforço de TI.

Não há base pública suficiente para classificar uma modalidade como "mais barata" no CNPJ da operação sem a proposta comercial.

## 6. Auttar TEF — escopo, interfaces e documentação

### 6.1 Conceitos de arquitetura

TEF = Transferência Eletrônica de Fundos no contexto de pagamento presencial integrado. O aplicativo comercial inicia a cobrança, comunica-se com um componente Auttar CTFClient e acompanha o processo do pinpad/terminal e da adquirência. Após retorno definitivo, confirma a operação no sistema e guarda comprovante, NSU, autorização, terminal, data, bandeira, valor e forma. O CTFClient não deve ser tratado como um endpoint e-commerce na Internet. A API Auttar e a API Getnet são produtos diferentes, ainda que a adquirente física possa ser Getnet se devidamente habilitada. [A1][A2][A3]

### 6.2 Interfaces publicadas oficialmente (verificar versão antes de programar)

A página de downloads oficial Auttar disponibiliza, na data pesquisada:

| Recurso / manual | Versão indicada no portal | Cenário a avaliar |
| --- | --- | --- |
| Manual de integração TEF-IP | v1.29 | Integração por protocolo TEF-IP e automação tradicional |
| Manual de integração DLL | v1.82 | Aplicativo Windows desktop chamando DLL, arquitetura 32/64 bits |
| Manual de integração WebSocket | v1.17 | Integração local via CTFClient WebSocket, potencial uso em navegador/ERP |
| Manual de integração SDK Android | v1.50 | App Android e hardware suportado, somente com kit/homologação |
| Manual de códigos de erro CTFClient | v1.04 | Tratamento de erros/retornos |
| Manual de integração POS TEF Postos | Disponível no portal | Segmento postos |
| CTFClient Windows | 4.33.4-0 | Instalador, portable 32/64 bits |
| Guia instalação CTFClient Java/Linux | Disponíveis | Ambientes alternativos |
| Pacotes de certificação | JRE 32 e 64 bits | Testes de integração |
| SDK Android | 4.16.3 Tap on Phone / 4.16.4 POS e pinpad | SDK, exemplos e documentação |

**Fonte primária para todos os manuais/downloads: https://auttar.com.br/downloads/**. O portal pode servir arquivos por scripts que não ficam expostos como URL estática indexável. Não reproduzir o conteúdo dos manuais no GitHub: usar os originais oficiais e obter autorização/licença quando necessário. O portal informa homologacao@auttar.com.br. [A1]

### 6.3 Escolha técnica por ambiente

**Windows PDV tradicional:** avaliar primeiro **DLL** se o sistema é executável e deseja controlar a interface do pagamento; ou **WebSocket** se o PDV é web local (ex.: XAMPP/Chrome) e o CTFClient oferece ponte local suportada. TEF-IP é alternativa para software com troca de arquivos/protocolo herdado. Não tentar controlar pinpad diretamente sem SDK homologado.

**Sistema web remoto:** não expor CTFClient, WebSocket ou DLL na internet. Usar um componente/agente local por estação, com autorização da Auttar, comunicação protegida ao servidor e retorno do status; navegador não deve enviar diretamente dados sensíveis para serviços externos não previstos.

**Android/SmartPOS:** escolher SDK Android somente após confirmar modelo de hardware, versão do pacote, fabricante, pinpad, fluxo financeiro, recursos habilitados e exigências de certificação. O SDK da Auttar não deve ser confundido com deeplink de pagamento do Get Smart/SmartPOS Getnet: este último é outra plataforma, com documentação própria, inclusive payment v3, Pix e callerId. [A1][G14]

### 6.4 Operações e estados de TEF

Fluxo funcional recomendado:
1. PDV cria venda interna como PAYMENT_PENDING.
2. PDV inicia pedido de TEF com identificador estável, valor em centavos e tipo/parcelas autorizados.
3. CTFClient/pinpad conduz seleção de cartão, senha e autorização.
4. Aplicação trata retornos intermediários sem fechar a venda antecipadamente.
5. Ao aprovar, confirma/finaliza segundo protocolo Auttar e grava NSU, autorização, data, comprovante e terminal.
6. Em falha após a operadora eventualmente ter autorizado, entrar em **STATUS_INDETERMINADO**; consultar/corrigir com TEF/operadora em vez de repetir cegamente a cobrança.
7. Em cancelamento, respeitar operação e autorização previstas no pacote, distinguindo cancelamento real da operadora de mera reversão contábil interna.

A integração Auttar com Sankhya, atualizada em 29/07/2026, confirma uso de CTFClient WebSocket, operações de crédito à vista, crédito parcelado lojista/emissor, débito, voucher e cancelamento, bem como limitações de estorno parcial naquela integração. Isso é evidência de um **ERP específico**, não garantia universal da Auttar em todos os contratos. [A2]

### 6.5 Requisitos de instalação e certificação

- Definir adquirente habilitada, identificação de estabelecimento/loja, licença e quantidade de PDVs.
- Escolher interface com Auttar e obter kit compatível, manuais, exemplos, dados de homologação e pinpad suportado.
- Validar driver, porta COM/USB/rede, arquitetura 32/64 bits, permissões e requisitos de runtime.
- Instalar e configurar CTFClient conforme Auttar e executar inicialização/carga de parâmetros.
- Definir emissão, guarda e reimpressão de comprovantes e regras fiscais locais.
- Ensaiar transações com crédito/débito, parcelas, cancelamento, perda de conectividade, timeout, reapresentação e reinício.
- Solicitar evidência de homologação e liberação comercial antes de uso real.
- Registrar qual método de conciliação TEF está disponível (retorno local, extrato adquirente, arquivos, API financeira).
- Contato oficial de homologação: homologacao@auttar.com.br. Suporte comercial e distribuição pelo portal Auttar. [A1][A2][A3]

### 6.6 Limitações que NÃO foram confirmadas em fontes oficiais abertas

Não declarar como garantidos: estorno parcial Auttar, Pix no pinpad, múltiplas adquirentes simultâneas, homologação automática da Getnet por usar Auttar, endpoints de produção, portas locais, limites de transações, compatibilidade com qualquer modelo USB/NFC, SLA de notificação e disponibilidade offline. Tudo isso depende do manual específico e contrato.

## 7. Arquitetura proposta para AcquaVale / hotelaria

~~~text
[Loja online AcquaVale]                 [Recepção / Caixa / PDV Windows]
       |                                           |
       +-----------> [API de Pedidos] <------------+
                         |    |   |
                         |    |   +--- [Adaptador Auttar TEF]
                         |    |               |
                         |    |         [CTFClient + Pinpad]
                         |    |
                         |    +------- [Getnet Web Checkout]
                         |                     |
                         |                 [Webhook]
                         |
                         +---------- [Getnet Payment Link API]
                         |                     |
                         |            [Consulta de ordens/eventos
                         |             comprovadamente habilitados]
                         |
                         v
                 [Ledger de pagamentos]
                         |
                   pagamento validado
                         |
                         v
                  [Fila idempotente]
                         |
                      Vale Visitor
                         |
                       HikCentral
                         |
                  Catracas / acessos
~~~

Separar o domínio **pedido** de **cobrança**: um pedido pode ter diversas tentativas e apenas um pagamento definitivo válido. Não vincular o ingresso a um único retorno de navegador. O backend deve receber/validar a informação financeira e só então disparar a integração de visitantes/catracas.

**Cenários específicos:**
- Venda on-line regular: Web Checkout, pedido e identificador correlacionados, liberação após estado final elegível.
- Compra assistida por recepcionista/WhatsApp: gerar link de uso único e verificar resultado antes de cadastrar/liberar visitante.
- Boleto: informar "aguardando compensação"; não liberar catraca enquanto houver apenas boleto emitido.
- Pix: permitir confirmação ágil, mas verificar a transação na origem.
- Caixa presencial: lançar venda, iniciar Auttar, aguardar conclusão real, guardar NSU e transmitir evento interno de pagamento.
- Se a venda já existe no POS legado sem API, conciliar por código de pedido/recibo com a plataforma de integração de impressão/tela, sem presumir que isso equivale à confirmação da adquirência.
- Cancelamentos: reverter os acessos quando permitido e registrar trilha de auditoria.

### 7.1 Modelo mínimo de dados sugerido

O desenho exato deve respeitar os esquemas e prefixos existentes da aplicação, sem alterar o banco em produção por este estudo.

| Entidade | Campos mínimos |
| --- | --- |
| orders | order_id, reservation_id, amount_cents, currency, status, created_at |
| payment_attempts | attempt_id, order_id, provider (GETNET_LINK / GETNET_CHECKOUT / GETNET_API / AUTTAR), provider_reference, method, status, amount_cents, idempotency_key, started_at, finished_at |
| payment_events | event_id, attempt_id, external_event_id, event_type, payload_hash, received_at, processed_at, status |
| tef_transactions | attempt_id, terminal_id, nsu, authorization_code, brand, installments, receipt_reference, provider_status, confirmed_at |
| fulfillment_outbox | id, order_id, kind (ISSUE_TICKET / REVOKE_TICKET), dedupe_key, status, attempts, next_retry_at |
| reconciliation | provider, settlement_day, reference, expected_amount_cents, actual_amount_cents, variance_cents, reconciliation_status |

Criar índices únicos em dedupe_key, event ID (quando aplicável), NSU com contexto de terminal/data/adquirência, e order_id+regra de negócio para emissão. Guardar apenas informações necessárias; não persistir cartão em claro nem PIN/CVV.

### 7.2 Máquina de estados operacional

- CREATED -> PENDING_PAYMENT -> PAID_CONFIRMED -> FULFILLMENT_PENDING -> FULFILLED.
- PENDING_PAYMENT -> EXPIRED ou CANCELLED (sem liberação).
- Tentativa TEF pode passar por UNKNOWN_RECONCILIATION_REQUIRED depois de timeout.
- PAID_CONFIRMED -> REFUND_PENDING -> REFUNDED; revogação de ingresso segue regra de negócio e histórico.
- Nunca regredir um pedido definitivamente pago por webhook antigo; manter ordenação por evento, autoridade do provedor e trilha auditável.

## 8. Segurança, resiliência, LGPD e operação

1. Backend PHP/serviço guarda segredos fora do DocumentRoot; não comitar .env, IDs privados, certificados, dumps de cartões ou segredos no GitHub.
2. HTTPS válido em todos os callbacks; autenticação adequada ao produto (Web Checkout Basic; Global Webhooks conforme inscrição). Validar conteúdo e correlacionar IDs com o pedido; não confiar em "success_url".
3. Implementar idempotência para operações financeiras e deduplicação de webhook; jamais repetir cobrança no timeout sem consulta prévia.
4. Registrar hora UTC, status original, referência financeira e estado normalizado; mascarar dados pessoais e limitar retenção de payloads.
5. Criar reconciliação agendada para cobrir callback atrasado/perdido, com métricas, filas de erros e reenvio seguro.
6. Segregar direitos de cancelamento/estorno por papel; usar auditoria não editável para transações.
7. Respeitar LGPD para fotos, documento, telefone, reserva e acesso de hóspedes/visitantes; definir bases legais, retenção e mínimo necessário.
8. Não expor Auttar CTFClient/WebSocket na rede pública e não deixar o frontend autorizar pagamento/entrega sem backend.
9. Gerir homologação separadamente por estabelecimento (CNPJ/filial), adquirente, meio de pagamento, terminais e conta Getnet.

## 9. Matriz de testes e critérios de aceite

| Teste | Resultado esperado |
| --- | --- |
| Criar link Pix habilitado | Link válido, método exibido, referências persistidas |
| Criar link boleto habilitado | Boleto emitido se contrato permitir; pedido pendente |
| Criar link boleto sem habilitação | Erro tratado e explicado sem vender promessa falsa |
| Boleto emitido e não pago | Sem ingresso/QR e sem liberação nas catracas |
| Pix confirmado | Uma aprovação financeira e um único conjunto de ingressos |
| Callback duplicado | Sem duplicar venda, QR ou visitante |
| Callback chega após navegador fechar | Confirmação processada normalmente |
| Navegador abre success_url falsa | Sem liberação sem validação do provedor |
| Pagamento recusado / antifraude negado | Pedido não liberado, possibilidade de nova tentativa |
| Timeout Getnet | Estado pendente/indeterminado + consulta segura |
| Estorno total/ parcial | Aplicar regra D+0/D+1 e produto contratado |
| Crédito parcelado | Valor, juros, parcelas e recibo corretos |
| Cartão offline/queda USB pinpad | TEF indeterminado controlado, sem nova cobrança automática |
| TEF autorizado e PDV reinicia | Conciliação/recuperação sem cobrança duplicada |
| Reimpressão comprovante TEF | Mesmo NSU e pagamento, sem refazer venda |
| Pedido com Web Checkout e Link simultâneos | Uma venda efetiva; evitar duas aprovações não detectadas |
| Webhook indisponível 72h | Recuperação pela consulta/reconciliação com alerta |
| Troca de CNPJ/credenciais | Nenhuma mistura de seller, checkout ou saldo |

**Pronto para produção** somente após: contrato assinado, sandbox validado, teste ponta-a-ponta de todos os meios, caso real de estorno, conciliação de recebíveis e aprovação Auttar/Getnet aplicáveis.

## 10. Perguntas objetivas para Getnet e Auttar

### Para Getnet

1. Nosso CNPJ possui **Payment Link API**, Link Get Pay manual, Web Checkout e Global API? Quais produtos/tenants, versões de documentação e credenciais correspondem a cada modalidade?
2. **Boleto no link:** está ativo para nosso CNPJ e nesta API? Qual banco emissor? Exige convênio Santander? Funciona também em link feito manualmente no Minha Conta?
3. O boleto do Payment Link apresenta linha digitável/PDF/QR? Como recebe status de boleto pago, vencido e liquidado? Há webhook ou só consulta de ordens? Qual SLA?
4. Existe webhook para pagamento via link com o mesmo mecanismo da Global API? Quais eventos, estrutura, autenticação e tentativas?
5. Quais as cobranças exatas de: link, transação negada, boleto emitido, Pix, antifraude, parcelado, conciliação e estorno?
6. Qual alternativa recomendada para e-commerce PHP próprio: Web Checkout redirect/iframe/lightbox ou Global API? Há oferta legada distinta em operação?
7. Qual documento OpenAPI/sandbox, endpoint de produção, seller ID e conjunto de headers oficialmente exigidos para nossa conta?
8. Como são realizados pré-autorização, captura, estorno, estorno parcial, 3DS e cofre nos produtos contratados?
9. Qual prazo de crédito por método e limite de reembolso? Como obter extrato financeiro ou relatório/API de conciliação?
10. Há integração recomendada para a Getnet adquirente presencial via Auttar e como são separados os estabelecimentos/terminais?

### Para Auttar

1. Qual interface homologável para um PDV Windows e um sistema web XAMPP: DLL v1.82, WebSocket v1.17 ou TEF-IP v1.29?
2. Quais versões do CTFClient e do pinpad/driver são homologadas, e qual modo de operação é suportado pela adquirente Getnet?
3. Quais modelos de pinpad e, se Android, SmartPOS/SDK são aceitos?
4. Quais operações exatas estão disponíveis: crédito, débito, voucher, Pix, parcelamento, pré-autorização, cancelamento, estorno total/parcial e reimpressão?
5. Como obter retorno pós-crash/timeout para saber se a adquirência aprovou a cobrança antes de repetir?
6. Qual fluxo formal de confirmação/finalização, cancelamento técnico e impressão do comprovante?
7. Há emulador de testes, credenciais de homologação, checklist de certificação e evidência de aprovação?
8. Quais preços, licenças por terminal, suporte, conectividade e SLA?
9. Há interface de conciliação ou extrato por NSU/terminal/data?
10. É permitido um agente local chamar WebSocket/DLL sem apresentar portas internas à internet?

## 11. Plano de implantação proposto

**Fase 0 — Contrato e habilitação:** obter confirmação escrita de boleto Payment Link, Pix, webhooks, tarifas, credenciais/sandbox e SDK Auttar correto. Sem esta fase, nada de prometer meio de pagamento.

**Fase 1 — Backend financeiro comum:** criar domínio de orders, payment_attempts, events, reconciliation e fulfillment_outbox; não mudar fluxo de ingressos existente sem testes.

**Fase 2 — Web Checkout:** integrar sandbox, checkout, webhook, consulta e conciliação; testar aprovação/recusa/duplicata antes de emitir ingressos.

**Fase 3 — Payment Link:** cobranças assistidas, link de uso único por pedido, boleto/Pix, consulta de ordens e confirmação financeira de ponta a ponta.

**Fase 4 — Auttar TEF:** instalar CTFClient no caixa piloto, integrar via interface definida oficialmente, testar reconciliação e equipamentos reais; expandir para demais caixas.

**Fase 5 — Operação e auditoria:** dashboard único de vendas, pagamentos pendentes, aprovações, duplicatas, divergências, estornos e envio ao Visitor/HikCentral, com alertas.

## 12. Fontes e documentação de referência

As URLs abaixo são pontos oficiais consultados em 06/11/2026 ou exemplos de terceiros assinalados. Podem mudar; conferir versões, país e conta.

### Getnet (fontes primárias)

- **[G1] Payment Link API — visão geral:** https://docs.globalgetnet.com/en/products/online-payments/payment-link-api
- **[G2] Payment Link — meios de pagamento por país:** https://docs.globalgetnet.com/en/products/online-payments/payment-link-api?doc=plk-supported-payment-methods
- **[G3] Página comercial Brasil — Link de Pagamento:** https://site.getnet.com.br/link-de-pagamento/
- **[G4] Criar link por API:** https://docs.globalgetnet.com/en/products/online-payments/payment-link-api?doc=plk-how-to-create-a-payment-link
- **[G5] Autenticação da Payment Link API:** https://devportalgetnet.sensedia-eng.com/en/products/online-payments/payment-link-api?doc=plk-authentication
- **[G6] Configurar link, meios e parcelas:** https://docs.globalgetnet.com/en/products/online-payments/payment-link-api?doc=plk-configure-the-payment-link
- **[G7] Guia Web Checkout / criar payment intent:** https://docs.globalgetnet.com/en/products/online-payments/web-checkout?doc=quickstart-create-a-payment-wbc
- **[G8] Global API — webhooks (autenticação, eventos, códigos):** https://docs.globalgetnet.com/en/products/online-payments/regional-api?doc=webhook-how-it-works
- **[G9] Web Checkout — webhook:** https://docs.globalgetnet.com/en/products/online-payments/web-checkout?doc=receive-webhook-payment-data-wbc
- **[G10] Getnet e-commerce Brasil / recursos comerciais:** https://site.getnet.com.br/parcerias/ecommerce/
- **[G11] Web Checkout API/Global API e operações:** https://docs.globalgetnet.com/en/products/online-payments/web-checkout/swagger
- **[G12] PDF Getnet sobre pacote de e-commerce de 2025 (apenas evidência histórica e condicional):** https://site.getnet.com.br/wp-content/uploads/2025/05/pacote_ecommerce_basico_21_05_2025.pdf
- **[G13] Payment Link — cancelamentos/estornos:** https://docs.globalgetnet.com/en/products/online-payments/payment-link-api?doc=plk-how-to-refund-a-payment
- **[G14] Get Smart (SmartPOS) — pagamento v3 por deeplink:** https://getstore.getnet.com.br/docs/integracao-pagamento/pagamento-deeplink/
- **[G15] Global API — detalhes de reembolso:** https://docs.globalgetnet.com/en/products/online-payments/regional-api?doc=refund-payment
- **[G16] Web Checkout — configuração por portal:** https://docs.globalgetnet.com/en/products/online-payments/web-checkout?doc=configuration-by-portal-wbc
- **[G17] Global API — referência de eventos:** https://docs.globalgetnet.com/en/products/online-payments/regional-api?doc=webhooks-reference

### Auttar (fonte primária e exemplos de integração externa)

- **[A1] Auttar — downloads oficiais: DLL v1.82, TEF-IP v1.29, WebSocket v1.17, SDK Android v1.50, CTFClient 4.33.4-0 e kits:** https://auttar.com.br/downloads/
- **[A2] Exemplo de ERP Sankhya com CTFClient WebSocket (não é especificação oficial genérica Auttar):** https://ajuda.sankhya.com.br/hc/pt-br/articles/360045109873-Integra%C3%A7%C3%A3o-com-TEF-AUTTAR
- **[A3] Exemplo de instalação de Auttar DLL em software terceiro (não constitui certificação geral):** https://kb.beemore.com/dc/intelidata/pt-br/domains/suporte/documents/integracao-com-tef-auttar-via-dll
- **[A4] Contato oficial para homologação (na página [A1]):** homologacao@auttar.com.br

## 13. Limites e validações pendentes

Este documento foi construído a partir de documentação pública e exemplos de integração. Não houve acesso ao portal autenticado do estabelecimento, credenciais, contratos ou ambiente de homologação; logo, **não foram validados pagamentos reais, endpoints específicos da conta, disponibilidade efetiva de boleto/TEF, preços finais nem webhooks em produção**. A documentação Getnet utiliza terminologias/versões diferentes para ofertas nacionais e globais; não transportar endpoints, campos e regras de uma família de API para outra sem conferir o OpenAPI vigente.

**Regra prática:** primeiro obter confirmação comercial/operacional por escrito; depois homologar tecnicamente; só então integrar o controle de acesso do parque e do hotel.

---
**Histórico:** v1.0 — 06/11/2026 — estudo inicial, comparação e plano de homologação.

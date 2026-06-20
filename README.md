# 🏦 Soma (Gestão Financeira a Dois)

O **Soma** é um aplicativo web focado na gestão financeira compartilhada para casais. Projetado com foco na experiência mobile, ele funciona como uma SPA (*Single Page Application*) com visual nativo de aplicativo (estilo iOS)[cite: 1].

O grande diferencial do projeto é sua arquitetura híbrida: ele roda de forma 100% instantânea salvando os dados no dispositivo através do `localStorage`[cite: 1] e utiliza uma planilha do **Google Sheets** como banco de dados em nuvem gratuito e sem servidor (Serverless), permitindo que duas pessoas mantenham as finanças sincronizadas em tempo real em aparelhos diferentes.

---

## ✨ Funcionalidades Implementadas

### 👫 Perfil e Configuração Local
*   **Identificação Isolada:** Cada usuário escolhe no seu próprio aparelho se é a Pessoa A ou a Pessoa B e personaliza os nomes (ex: Rodrigo e Jacqueline)[cite: 1].
*   **Armazenamento de Credenciais:** O link secreto da planilha do Google Sheets é configurado e guardado diretamente na aba de configurações do aparelho, mantendo o código público no GitHub seguro e sem chaves expostas.

### 💰 Cofres Virtuais com Metas Individuais
*   **Criação de Cofres:** Organize as economias criando cofres com nomes e símbolos (emojis) personalizados[cite: 1].
*   **Barras de Progresso:** Defina metas financeiras em cada cofre. O app exibirá uma barra de progresso visual indicando a porcentagem já atingida para o objetivo.
*   **Histórico de Atividade:** Controle total de depósitos, retiradas e pagamentos recentes cruzando as informações com quem realizou a ação[cite: 1].

### 🤝 Empréstimos Internos com Carência
*   **Retirada Planejada:** Permite retirar dinheiro de um cofre específico simulando um empréstimo com parcelamento (ex: 2x, 3x, 6x)[cite: 1].
*   **Período de Carência:** Opção de agendar o início do pagamento para meses futuros (daqui a 1 ou 2 meses), exibindo alertas de carência ativa na tela de detalhes.

### 📊 Gestão Mensal Inteligente e Rateio
*   **Fluxo de Entrada:** O mês inicia limpo. A interface de contas só é liberada após você inserir os ganhos/saldos do período.
*   **Separação por Fonte:** Os saldos inseridos são divididos por categorias de uso: Dinheiro Físico, Vale-Alimentação e Conta Bancária[cite: 1].
*   **Rateio de Despesas:** Ao marcar uma conta como paga, o app sugere de forma inteligente a divisão do dinheiro (priorizando o uso do Vale-Alimentação e deixando o saldo bancário livre)[cite: 1].
*   **Despesas Direcionadas:** Possibilidade de cadastrar uma conta recorrente cujo dinheiro vá direto para um cofre (ex: retirar 60 euros todo mês da conta corrente e injetar automaticamente no cofre de eletrodomésticos).

### 🤖 Motor de Auto-Distribuição de Sobras
*   **Sobra Livre:** O app calcula automaticamente o valor exato que sobrou após a quitação de todas as despesas[cite: 1].
*   **Injeção Inteligente:** Um botão dinâmico analisa quais cofres possuem metas pendentes e distribui a sobra do mês de forma automática e proporcional entre eles até que as metas sejam batidas.

---

## 🚀 Como Configurar o Google Sheets como Banco de Dados

Para ativar a sincronização entre dois celulares sem custos, siga este guia:

### 1. Criar a Planilha Base
1. Acesse o [Google Sheets](https://sheets.google.com) e crie uma planilha em branco.
2. Dê um nome a ela (ex: `Backup_Soma`).
3. Deixe-a aberta. O aplicativo lerá e gravará tudo compactado na **célula A1** dessa planilha.

### 2. Configurar o Servidor (Google Apps Script)
1. No menu superior da planilha, clique em **Extensões** > **Apps Script**.
2. Apague qualquer código existente na tela e cole o bloco abaixo:

```javascript
function doPost(e) {
  try {
    var sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
    var data = e.postData.contents; 
    
    // Grava o JSON completo na célula A1
    sheet.getRange('A1').setValue(data);
    // Registra o horário do backup na célula B1
    sheet.getRange('B1').setValue("Última sincronização: " + new Date().toLocaleString("pt-BR"));
    
    return ContentService.createTextOutput(JSON.stringify({"status": "sucesso"}))
      .setMimeType(ContentService.MimeType.JSON);
  } catch (error) {
    return ContentService.createTextOutput(JSON.stringify({"erro": error.toString()}))
      .setMimeType(ContentService.MimeType.JSON);
  }
}

function doGet(e) {
  var sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  var data = sheet.getRange('A1').getValue(); 
  
  if (!data) {
    data = "{}"; 
  }
  
  return ContentService.createTextOutput(data)
    .setMimeType(ContentService.MimeType.JSON);
}

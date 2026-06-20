# 🏦 Soma (Gestão Financeira a Dois)

Um aplicativo web *Progressive Web App* (PWA) de página única, desenvolvido inteiramente em HTML, CSS e Vanilla JavaScript, focado na gestão financeira para casais. 

O app foi projetado para funcionar perfeitamente em dispositivos móveis, operando offline de forma instantânea via `localStorage` e sincronizando os dados em tempo real utilizando o **Google Sheets** como banco de dados gratuito e sem servidor (Serverless).

## ✨ Funcionalidades

*   👫 **Gestão Compartilhada:** Perfil para Pessoa A e Pessoa B, com rastreamento de quem fez cada movimentação.
*   💰 **Cofres com Metas:** Criação de cofres virtuais (ex: Viagem, Eletrodomésticos) com acompanhamento de metas em barra de progresso.
*   📊 **Gestão Mensal Inteligente:** Lançamento de rendas separadas por fontes (Dinheiro Físico, Vale-Alimentação e Conta Bancária).
*   🛒 **Rateio Inteligente:** Ao pagar contas, o app sugere o uso de saldos específicos (ex: prioriza gastar o vale-alimentação primeiro).
*   🤝 **Empréstimos com Carência:** Retirada de dinheiro dos cofres com parcelamento flexível e opção de iniciar o pagamento em meses futuros (carência).
*   🤖 **Distribuição Automática:** Motor que calcula a "Sobra Livre" do mês e permite distribuir o dinheiro excedente automaticamente para os cofres que ainda não atingiram a meta.

---

## 🚀 Como usar e configurar (Instalação)

Este aplicativo não precisa de um servidor (Node.js, PHP, etc) para rodar. Você hospeda a interface aqui no GitHub Pages e os dados ficam na sua conta pessoal do Google.

### Passo 1: Hospedar o App (Interface)
1. Faça um Fork ou faça o upload do arquivo `index.html` neste repositório.
2. Vá na aba **Settings** > **Pages**.
3. Em "Source", selecione a branch `main` e salve.
4. O GitHub vai gerar um link para o seu site (ex: `https://rodrig0almeida.github.io/Finan-as`).
5. Abra este link no Safari (iOS) ou Chrome (Android) e selecione **"Adicionar à Tela de Início"** para instalar como um aplicativo.

### Passo 2: Criar o Banco de Dados (Google Sheets)
Para que os celulares de ambas as pessoas sincronizem, usaremos o Google Sheets.

1. Acesse o [Google Sheets](https://sheets.google.com) e crie uma planilha em branco (ex: "BD_Soma").
2. No menu superior, clique em **Extensões** > **Apps Script**.
3. Apague todo o código e cole o código abaixo:

```javascript
function doPost(e) {
  try {
    var sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
    var data = e.postData.contents; 
    
    // Salva o banco de dados na célula A1
    sheet.getRange('A1').setValue(data);
    // Salva a data e hora do último sync na célula B1
    sheet.getRange('B1').setValue("Última sincronização: " + new Date().toLocaleString("pt-BR"));
    
    return ContentService.createTextOutput(JSON.stringify({"status": "sucesso"})).setMimeType(ContentService.MimeType.JSON);
  } catch (error) {
    return ContentService.createTextOutput(JSON.stringify({"erro": error.toString()})).setMimeType(ContentService.MimeType.JSON);
  }
}

function doGet(e) {
  var sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  var data = sheet.getRange('A1').getValue(); 
  
  if (!data) data = "{}"; 
  
  return ContentService.createTextOutput(data).setMimeType(ContentService.MimeType.JSON);
}

# 🏦 Soma (Gestão Financeira a Dois)

O **Soma** é um aplicativo web focado na gestão financeira compartilhada para casais. Projetado com foco na experiência mobile, ele funciona como uma SPA (*Single Page Application*) com visual nativo de aplicativo (estilo iOS).

O grande diferencial do projeto é sua arquitetura híbrida: ele roda de forma 100% instantânea salvando os dados no dispositivo através do `localStorage` e utiliza uma planilha do **Google Sheets** como banco de dados em nuvem gratuito e sem servidor (Serverless), permitindo que duas pessoas mantenham as finanças sincronizadas em tempo real em aparelhos diferentes.

---

## ✨ Funcionalidades Implementadas

### 👫 Perfil e Configuração Local
* **Identificação Isolada:** Cada usuário escolhe no seu próprio aparelho se é a Pessoa A ou a Pessoa B e personaliza os nomes.
* **Armazenamento Seguro:** O link secreto da planilha do Google Sheets é colado diretamente na aba de configurações do aplicativo no seu celular. O código-fonte fica público no GitHub, mas os seus dados ficam privados.

### 💰 Cofres Virtuais com Metas Individuais
* **Criação de Cofres:** Organize as economias criando cofres com nomes e símbolos (emojis) personalizados.
* **Barras de Progresso:** Defina metas financeiras em cada cofre. O app exibe o progresso em tempo real.
* **Histórico de Atividade:** Controle total de depósitos, retiradas e pagamentos recentes.

### 🤝 Empréstimos Internos com Carência
* **Retirada Planejada:** Retire dinheiro de um cofre específico simulando um empréstimo parcelado (ex: 2x, 3x, 6x).
* **Período de Carência:** Agende o início do pagamento para meses futuros (daqui a 1 ou 2 meses), exibindo um alerta vermelho nos detalhes do cofre.

### 📊 Gestão Mensal Inteligente e Rateio
* **Fluxo de Entrada:** Insira seus saldos separados por categorias: Dinheiro Físico, Vale-Alimentação e Conta Bancária.
* **Rateio de Despesas:** Ao pagar uma conta, o app sugere o rateio inteligente (ex: prioriza zerar o Vale-Alimentação antes de usar o dinheiro físico).
* **Despesas Direcionadas a Cofres:** Cadastre contas recorrentes cujo dinheiro pago seja transferido automaticamente para um cofre escolhido.

### 🤖 Motor de Auto-Distribuição de Sobras
* **Injeção Inteligente:** Um botão especial ("✨ Distribuir pelas metas") calcula o que sobrou livre no seu mês e injeta automaticamente esse dinheiro nos cofres que ainda não atingiram as metas.

---

## 🚀 Como Configurar o Banco de Dados (Google Sheets)

Para que os dois celulares conversem entre si, precisamos criar a planilha e **gerar o link de sincronização**. Siga o passo a passo com atenção:

### Passo 1: Criar a Planilha e Inserir o Código
1. Acesse o [Google Sheets](https://sheets.google.com) e crie uma planilha em branco (ex: `BD_Soma`).
2. No menu superior da planilha, clique em **Extensões** > **Apps Script**.
3. Apague todo o código que estiver na tela e cole este bloco abaixo:

```javascript
function doPost(e) {
  try {
    var sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
    var data = e.postData.contents; 
    
    sheet.getRange('A1').setValue(data);
    sheet.getRange('B1').setValue("Última sincronização: " + new Date().toLocaleString("pt-BR"));
    
    return ContentService.createTextOutput(JSON.stringify({"status": "sucesso"})).setMimeType(ContentService.MimeType.JSON);
  } catch (error) {
    return ContentService.createTextOutput(JSON.stringify({"erro": error.toString()})).setMimeType(ContentService.MimeType.JSON);
  }
}

function doGet(e) {
  var sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  var data = sheet.getRange('A1').getValue(); 
  if (!data) { data = "{}"; }
  return ContentService.createTextOutput(data).setMimeType(ContentService.MimeType.JSON);
}
```

4. Clique no ícone de **Salvar** (disquete) no menu superior.

### Passo 2: Como Gerar e Pegar o Link (Atenção Aqui!)
Este é o momento de criar a URL que o seu aplicativo vai usar.
1. No canto superior direito do Apps Script, clique no botão azul **Implantar** e escolha **Nova implantação**.
2. Clique no ícone de engrenagem (`⚙️`) ao lado de "Selecione o tipo" e escolha **App da Web**.
3. Preencha os campos **EXATAMENTE** desta forma:
   * **Descrição:** Pode escrever "Versão 1".
   * **Executar como:** *Você (seu-email@gmail.com)*
   * **Quem tem acesso:** Selecione **Qualquer pessoa** *(Se não marcar "Qualquer pessoa", o celular não vai conseguir salvar)*.
4. Clique no botão azul **Implantar**.
5. Vai aparecer um aviso de "Autorização necessária". Clique em **Autorizar acessos** e escolha o seu e-mail.
6. O Google mostrará uma tela dizendo "O Google não verificou este app". Clique na palavra **Avançado** (lá embaixo) e depois clique em **Acessar projeto sem título (não seguro)**. Clique em **Permitir**.
7. Na última tela que aparecer, você verá escrito "URL do app da Web" e um link gigante embaixo (terminando com `/exec`). **Copie este link gigante.**

### Passo 3: Colar o Link no Aplicativo
1. Abra o site do seu aplicativo (o link do GitHub Pages) no seu celular.
2. Navegue até a aba inferior direita chamada **Você** (Configurações).
3. Procure o campo **"URL de Sincronização (Google Sheets)"** e cole o link gigante lá dentro.
4. Defina se você é a Pessoa A ou Pessoa B.
5. Pronto! Faça o mesmo no celular da sua dupla e vocês já estarão sincronizados.

---

## 🛠️ Performance e Sincronização Manual
Para não estourar o limite de acessos diários do Google:
* O app salva localmente e só envia dados para a nuvem 2 segundos após você terminar de fazer uma alteração.
* Ele puxa dados novos automaticamente apenas quando o aplicativo é aberto.
* **Quer puxar atualizações manualmente?** Basta tocar na pílula escrita "sincronizado" ou "offline" no topo do app para forçar a atualização da tela naquele momento.

---

## 📱 Dica: Instale no Celular como um App Real
Abra o site no navegador do celular (Safari ou Chrome), clique no botão de **Compartilhar** do navegador e escolha **"Adicionar à Tela de Início"**. O navegador ocultará as barras de endereço e o ícone ficará junto com seus outros aplicativos.

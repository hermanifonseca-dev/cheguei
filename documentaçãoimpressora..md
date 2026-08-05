# 🏷️ Label Editor Engine (Elgin L42 Pro Full Compatible)

Este projeto consiste no desenvolvimento de um editor de etiquetas visual (WYSIWYG) capaz de exportar layouts diretamente em comandos nativos de impressoras térmicas. O foco principal da homologação de hardware é a impressora **Elgin L42 Pro Full**.

---

## 🏗️ 1. Arquitetura do Sistema

Para garantir performance e nitidez perfeita (sem serrilhados), o editor **não envia imagens/prints** para a impressora. O fluxo transforma elementos visuais em código de máquina (Strings textuais) enviados em modo **RAW**.



### Componentes Principais:
1. **Designer Canvas:** Interface do usuário onde elementos (textos, códigos de barras, QR codes e linhas) são posicionados tridimensionalmente.
2. **Compiler Engine:** Módulo responsável por varrer a árvore de elementos do Canvas e traduzir as coordenadas de pixels para coordenadas de pontos térmicos (`dots`) da linguagem alvo.
3. **Hardware Communicator:** Módulo de IO responsável por despachar a string bruta de comandos para a impressora via Rede (TCP/IP), USB Local ou Bluetooth.

---

## 📐 2. Modelo de Dados e Conversão Matriz (Mapeamento de Coordenadas)

A cabeça de impressão da Elgin L42 Pro Full trabalha com uma resolução fixa de **203 DPI (dots per inch / pontos por polegada)**. O editor trabalha internamente com milímetros (mm) para exibição e faz a conversão direta para pontos (unidade aceita pelo hardware).

### Fórmula de Conversão:
$$Pontos (dots) = \left( \frac{\text{Medida em mm}}{25.4} \right) \times 203$$

### Tabela de Referência Rápida (Para o Canvas):

| Dimensão Física (mm) | Equivalência em Pontos (dots) | Pixels em Tela (Escala 1:1 aproximada) |
| :--- | :--- | :--- |
| **1 mm** | ~8 pontos | 8 px |
| **10 mm (1 cm)** | ~80 pontos | 80 px |
| **100 mm (Largura Max)** | 800 pontos | 800 px |

### JSON Schema Sugerido para o Layout da Etiqueta:
```json
{
  "canvas": {
    "width_mm": 100,
    "height_mm": 50,
    "resolution_dpi": 203
  },
  "elements": [
    {
      "id": "txt_01",
      "type": "text",
      "x_mm": 5,
      "y_mm": 10,
      "content": "PRODUTO MODELO",
      "font_size_pt": 12
    },
    {
      "id": "bar_01",
      "type": "barcode_128",
      "x_mm": 5,
      "y_mm": 25,
      "content": "7891234567890",
      "height_mm": 15
    }
  ]
}
```

---

## 🔤 3. Linguagem Alvo de Impressão: ZPL II

Apesar de a Elgin L42 Pro Full aceitar PPLA e PPLB, a **ZPL II (Zebra Programming Language)** foi adotada devido à vasta gama de emuladores e suporte a fontes escaláveis. 

### Mapeamento de Comandos Essenciais

Todo arquivo gerado deve seguir a estrutura de bloco delimitada por `^XA` (Início) e `^XZ` (Fim).

| Tipo de Elemento | Comando ZPL | Exemplo de Implementação |
| :--- | :--- | :--- |
| **Configuração de Largura** | `^PW[pontos]` | `^PW800` (Define largura de 100mm) |
| **Posicionamento de Campo** | `^FO[X],[Y]` | `^FO50,100` (Posiciona elemento em X=50, Y=100) |
| **Texto Padrão** | `^A[fonte][direção],[alt],[lar]` | `^A0N,30,30^FDTexto Exemplo^FS` |
| **Código de Barras (Code 128)** | `^BC[dir],[alt],[line],[text]` | `^BCN,80,Y,N,N^FD7891234^FS` |
| **QR Code** | `^BQA,[mod],[mult]` | `^BQN,2,6^FDQA,Conteudo do QR^FS` |
| **Linhas / Caixas** | `^GB[lar],[alt],[esp]` | `^GB400,0,4^FS` (Desenha uma linha horizontal de 400 pontos) |

*Nota: Todo comando de dado em ZPL precisa terminar com o delimitador de campo `^FS` (Field Separator).*

---

## 🖥️ 4. Métodos de Comunicação com a Elgin L42 Pro Full

A transferência do arquivo final gerado pelo Compiler Engine para o hardware deve ser feita sem filtros de sistema operacional.

### Opção A: Impressão via Rede (Ethernet) - Altamente Recomendada
A Elgin L42 Pro Full possui placa de rede nativa. A comunicação é feita injetando a string ZPL diretamente na porta padrão **9100** usando Sockets TCP de baixo nível.

* **Exemplo conceitual (Node.js / Python):**
```javascript
const net = require('net');
const client = new net.Socket();

const zplString = "^XA^PW800^FO50,50^A0N,40,40^FDElgin L42 Pro Full^FS^XZ";

client.connect(9100, '192.168.1.100', () => {
    client.write(zplString);
    client.destroy();
});
```

### Opção B: Impressão USB Local (Spooler do S.O.)
Se conectada via USB, o driver padrão do Windows vai tentar interpretar o arquivo como imagem. Deve-se forçar o envio em modo **RAW**.
* **No Windows:** Utilizar a biblioteca nativa `winspool.drv` chamando as rotinas `OpenPrinter`, `StartDocPrinter` com a flag de tipo de dados configurada como `"RAW"`.
* **Ambiente Web (Browser):** Navegadores não acessam portas USB locais diretamente por motivos de segurança. É necessário um utilitário local (Ex: QZ Tray, JSPrintManager) instalado na máquina do cliente agindo como ponte Websocket entre a aplicação web e o Spooler USB.

---

## 🛠️ 5. Ferramentas e Recursos de Apoio ao Desenvolvimento

Para testar a lógica do compilador sem precisar gastar ribbon e papel fisicamente a cada alteração de código:

* **Labelary API (Emulador ZPL):** Envie via POST HTTP a string ZPL gerada pelo seu editor para `http://labelary.com` e receba de volta um PNG exato de como a etiqueta será impressa na Elgin L42 Pro Full. Excelente para renderizar o "Live Preview" dentro do próprio software.
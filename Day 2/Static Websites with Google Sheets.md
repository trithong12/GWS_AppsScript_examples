<h1 style="display: flex; justify-content: space-between; align-items: center;">
    Static Websites with Google Sheets
    <img src="CloudMile-logo.gif" alt="GIF" style="height: 2em;" />
</h1>

- [Google Sheets spreadsheet data](#google-sheets-spreadsheet-data)
- [Main functions](#main-functions)
- [Website UI HTML](#website-ui-html)
---
### Google Sheets spreadsheet data
| Branch Name  | Address                              | Contact Number |
|--------------|--------------------------------------|----------------|
| 臺北101店     | 臺北市信義區信義路5段7號                  | 02-1234-5678   |
| 臺北車站店    | 臺北市中正區忠孝西路1段49號               | 02-1234-5679   |
| 淡水店       | 新北市淡水區中正路1號                    | 02-8765-4321   |

---

### Main functions
<b>Remember to configure a Script Property with key "SPREAD_SHEET_ID", its value would be your actual owned spreadsheet ID.</b>

<em>main.gs</em>
```javascript
function doGet() {
  return HtmlService.createHtmlOutputFromFile('index');
}

function getStoreData() {
  const spreadSheetId = PropertiesService.getScriptProperties().getProperty("SPREAD_SHEET_ID");
  const sheet = SpreadsheetApp.openById(spreadSheetId).getSheetByName('StoreBranches');
  const data = sheet.getDataRange().getValues();
  const headers = data.shift(); // Remove headers
  const storeData = data.map(row => ({
    branchName: row[0],
    address: row[1],
    contactNumber: row[2]
  }));
  return storeData;
}
```
---
### Website UI HTML
<em>index.html</em>
```html
<!DOCTYPE html>
<html>
<head>
  <base target="_top">
  <style>
    body {
      font-family: Arial, sans-serif;
      margin: 20px;
    }
    .store-card {
      border: 1px solid #ccc;
      padding: 15px;
      margin-bottom: 10px;
      border-radius: 5px;
    }
    h2 {
      color: #333;
    }
  </style>
  <script>
    function fetchStoreData() {
      google.script.run.withSuccessHandler(displayStoreData).getStoreData();
    }
    
    function displayStoreData(data) {
      const container = document.getElementById('storeContainer');
      container.innerHTML = '';  // Clear previous content
      
      data.forEach(store => {
        const card = document.createElement('div');
        card.className = 'store-card';
        card.innerHTML = `
          <h3>${store.branchName}</h3>
          <p><strong>Address:</strong> ${store.address}</p>
          <p><strong>Contact Number:</strong> ${store.contactNumber}</p>
        `;
        container.appendChild(card);
      });
    }
    
    window.onload = fetchStoreData;
  </script>
</head>
<body>
  <h2>分店資訊</h2>
  <div id="storeContainer"></div>
</body>
</html>
```
<h1 style="display: flex; justify-content: space-between; align-items: center;">
    News Summarization with Gemini and Google Sheets
    <img src="CloudMile-logo.gif" alt="GIF" style="height: 2em;" />
</h1>

- [Google Sheets spreadsheet columns](#google-sheets-spreadsheet-columns)
- [Fetch News](#fetch-news)
- [Define a Google Sheets function - call Gemini](#define-a-google-sheets-function---call-gemini)
- [Advanced revision](#advanced-revision)
---
### Google Sheets spreadsheet columns
| Subject | URL | 摘要 | 新聞分類 |
|---------|-----|------|---------|
|         |     | 請幫我用5句以內的繁體中文摘要這篇新聞。 | 請幫我分類此新聞。輸出給我簡單中文分類結果就好，不用長篇大論解釋，例如：政治、科技。 |

---

### Fetch News
<em>fetch-news.gs</em>
```javascript
/**
 * Main function to fetch news and write them to Google Sheets.
 */
function fetchAndWriteNews() {
  const url = 'https://tw.news.yahoo.com/';
  const html = UrlFetchApp.fetch(url).getContentText();
  
  // Parse the HTML and extract top 10 news
  const newsData = parseNews(html);
  
  // Open the target Google Sheet
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  
  // Write each news item to the sheet
  newsData.slice(0, 5).forEach((news, index) => {
    const title = news.title;
    const link = news.url;
    
    sheet.getRange(index+3, 1, 1, 2).setValues([[title, link]])
    Utilities.sleep(1000); // Sleep to avoid hitting rate limits
  });
}

/**
 * Parse the news titles and links from the specific HTML structure.
 */
function parseNews(html) {
  const newsItems = [];
  
  // Regex to extract <ul> block with class 'H(100%) D(ib) Mstart(24px) W(32.7%)'
  const ulRegex = /<ul class="H\(100%\) D\(ib\) Mstart\(24px\) W\(32\.7%\)">([\s\S]*?)<\/ul>/g;
  let ulMatch;

  // Loop through each <ul> block
  while ((ulMatch = ulRegex.exec(html)) !== null) {
    const ulContent = ulMatch[1]; // Extract the content inside the <ul> tag

    // Regex to extract each <a> tag inside the <ul>
    const aRegex = /<a[^>]+href="([^"]+)"[^>]*>(.*?)<\/a>/g;
    let aMatch;

    // Loop through each <a> tag inside the <ul>
    while ((aMatch = aRegex.exec(ulContent)) !== null) {
      const url = aMatch[1]; // Append base URL if needed
      const title = aMatch[2].replace(/<[^>]+>/g, ''); // Clean HTML tags from the title

      // Add to the newsItems array if the link and title are valid
      if (title.trim() !== '') {
        newsItems.push({ title, url });
      }
    }
  }

  return newsItems;
}

```

---

### Define a Google Sheets function - call Gemini
<b>Reminder: Configure a Script Property with key "GEMINI_API_KEY", its value would be your actual owned Gemini API Key.</b>

<em>call-gemini.gs</em>
```javascript
const GEMINI_API_KEY = PropertiesService.getScriptProperties().getProperty("GEMINI_API_KEY"); // Replace with your Gemini API key
const GEMINI_API_URL = `https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash-latest:generateContent?key=${GEMINI_API_KEY}`; // Example endpoint

/**
 * Sends a prompt containing a specified range of data to the Gemini API and retrieves a summary.
 *
 * @param {string} range - A reference to a range of data, typically in a Google Sheets context.
 * @param {string} prompt - The user-specified instruction that guides the summary generation.
 * 
 * @returns {string} The output generated by the Gemini API or an error/fallback message.
 */
function gemini(range, prompt) {
  prompt = `For the range of ${range}, ${prompt}.`;
  const payload = JSON.stringify({
    "contents": [{
      "parts": [{
        "text": prompt
      }]
    }]
  });

  const options = {
    method: 'POST',
    contentType: 'application/json',
    payload
  };
  
  try {
    const response = UrlFetchApp.fetch(GEMINI_API_URL, options);
    const json = JSON.parse(response.getContentText());

    return json.candidates[0].content.parts[0].text || 'Prompt response unavailable.';
  } catch (error) {
    Logger.log(`Error: ${error}`);
    return 'Error retrieving prompt response.';
  }
}
```
---

### Advanced revision

Simply set up a trigger to daily call the setupAndProcessNews() function, and it will:
- Create a new sheet named with the current date.
- Populate it with the fetched news data.
- Apply the formulas to generate summaries and news classifications using the gemini function.

```javascript
/**
 * Creates a new sheet with the current date, sets up headers and prompts, and processes news data.
 */
function setupAndProcessNews() {
  // Create a new sheet with the current date as the sheet name
  const spreadsheet = SpreadsheetApp.getActiveSpreadsheet();
  const date = new Date();
  const sheetName = Utilities.formatDate(date, Session.getScriptTimeZone(), 'yyyy-MM-dd');
  const newSheet = spreadsheet.insertSheet(sheetName);
  
  // Set headers (Row 1)
  newSheet.getRange('A1:D1').setValues([['Subject', 'URL', '摘要', '新聞分類']]);
  
  // Set prompts (Row 2)
  newSheet.getRange('A2:D2').setValues([[
    '', '', '請幫我用5句以內的繁體中文摘要這篇新聞。', 
    '請幫我分類此新聞。輸出給我簡單中文分類結果就好，不用長篇大論解釋，例如：政治、科技。'
  ]]);

  // Fetch and write news data to the new sheet
  fetchAndWriteNewsToSheet(newSheet);
  
  // Apply the gemini formula to generate summaries and categories for the news items
  for (let i = 3; i <= 7; i++) { // Assuming we're working with 5 news items
    newSheet.getRange(`C${i}`).setFormula(`=gemini(B${i}, $C$2)`);
    newSheet.getRange(`D${i}`).setFormula(`=gemini(B${i}, $D$2)`);
  }
  
  Logger.log(`New sheet '${sheetName}' created and populated.`);
}

/**
 * Fetches the top 5 news items and writes them to the provided sheet.
 */
function fetchAndWriteNewsToSheet(sheet) {
  const url = 'https://tw.news.yahoo.com/';
  const html = UrlFetchApp.fetch(url).getContentText();
  
  // Parse the HTML and extract top 10 news
  const newsData = parseNews(html);
  
  // Write each news item to the sheet (limit to top 5)
  newsData.slice(0, 5).forEach((news, index) => {
    const title = news.title;
    const link = news.url;
    
    // Write the title and link to columns A and B
    sheet.getRange(index + 3, 1, 1, 2).setValues([[title, link]]);
    Utilities.sleep(1000); // Sleep to avoid hitting rate limits
  });
}
```
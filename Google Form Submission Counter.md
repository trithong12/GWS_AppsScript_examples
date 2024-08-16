<h1 style="display: flex; justify-content: space-between; align-items: center;">
    Google Form Submission Counter
    <img src="CloudMile-logo.gif" alt="GIF" style="height: 2em;" />
</h1>

```javascript
/**
 * Tracks the number of form submissions by a user and updates the count in both User Properties 
 * and a specific sheet. This function is triggered when a form response is submitted.
 *
 * @param {Object} e - The event object containing information about the form submission, 
 *                     including the range where the form response is written.
 */
function trackUserSubmissions(e) {
  const rowNumber = e.range.getRow();
  const userEmail = e.range.getSheet().getSheetValues(rowNumber, 2, 1, 1)[0][0];
  var userProperties = PropertiesService.getUserProperties();
  var submissionCount = userProperties.getProperty("submissionCount_" + userEmail);
  
  if (!submissionCount) {
    submissionCount = 0;
  }
  
  submissionCount = parseInt(submissionCount) + 1;
  userProperties.setProperty("submissionCount_" + userEmail, submissionCount);
  
  Logger.log("User " + userEmail + " has submitted the form " + submissionCount + " times.");
  
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName("Submission Count");
  const userCountRecordRowNumber = findRowNumber(sheet, 1, userEmail);
  if (userCountRecordRowNumber == -1) {
    sheet.appendRow([userEmail, submissionCount])
  } else {
    sheet.getRange(userCountRecordRowNumber, 2).setValue(submissionCount);
  }
}

/**
 * Finds the row number in the specified sheet that contains a given value in a specific column.
 *
 * @param {Sheet} sheet - The Google Sheet object to search within.
 * @param {string} searchValue - The value to search for within the specified column.
 * @return {number} The row number where the value is found (1-based index), or -1 if not found.
 */
function findRowNumber(sheet, column, searchValue) {
  var range = sheet.getRange(1, column, sheet.getLastRow(), 1).getValues();
  
  for (var i = 0; i < range.length; i++) {
    if (range[i][0] == searchValue) {
      return i + 1;
    }
  }
  
  return -1;
}
```
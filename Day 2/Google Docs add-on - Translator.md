<h1 style="display: flex; justify-content: space-between; align-items: center;">
    Google Docs add-on - Translator
    <img src="CloudMile-logo.gif" alt="GIF" style="height: 2em;" />
</h1>

Reference: [https://developers.google.com/workspace/add-ons/editors/docs/quickstart/translate](https://developers.google.com/workspace/add-ons/editors/docs/quickstart/translate)

---

- [Apps Script functions](#apps-script-functions)
- [Add-on UI](#add-on-ui)
---

### Apps Script functions
<em>translate.gs</em>
```javascript
/**
 * @OnlyCurrentDoc
 *
 * The above comment directs Apps Script to limit the scope of file
 * access for this add-on. It specifies that this add-on will only
 * attempt to read or modify the files in which the add-on is used,
 * and not all of the user's files. The authorization request message
 * presented to users will reflect this limited scope.
 */

/**
 * Creates a menu entry in the Google Docs UI when the document is opened.
 * This method is only used by the regular add-on, and is never called by
 * the mobile add-on version.
 *
 * @param {object} e The event parameter for a simple onOpen trigger. To
 *     determine which authorization mode (ScriptApp.AuthMode) the trigger is
 *     running in, inspect e.authMode.
 */
function onOpen(e) {
  DocumentApp.getUi().createAddonMenu()
      .addItem('Start', 'showSidebar')
      .addToUi();
}

/**
 * Runs when the add-on is installed.
 * This method is only used by the regular add-on, and is never called by
 * the mobile add-on version.
 *
 * @param {object} e The event parameter for a simple onInstall trigger. To
 *     determine which authorization mode (ScriptApp.AuthMode) the trigger is
 *     running in, inspect e.authMode. (In practice, onInstall triggers always
 *     run in AuthMode.FULL, but onOpen triggers may be AuthMode.LIMITED or
 *     AuthMode.NONE.)
 */
function onInstall(e) {
  onOpen(e);
}

/**
 * Opens a sidebar in the document containing the add-on's user interface.
 * This method is only used by the regular add-on, and is never called by
 * the mobile add-on version.
 */
function showSidebar() {
  const ui = HtmlService.createHtmlOutputFromFile('sidebar')
      .setTitle('Translate');
  DocumentApp.getUi().showSidebar(ui);
}

/**
 * Gets the text the user has selected. If there is no selection,
 * this function displays an error message.
 *
 * @return {Array.<string>} The selected text.
 */
function getSelectedText() {
  const selection = DocumentApp.getActiveDocument().getSelection();
  const text = [];
  if (selection) {
    const elements = selection.getSelectedElements();
    for (let i = 0; i < elements.length; ++i) {
      if (elements[i].isPartial()) {
        const element = elements[i].getElement().asText();
        const startIndex = elements[i].getStartOffset();
        const endIndex = elements[i].getEndOffsetInclusive();

        text.push(element.getText().substring(startIndex, endIndex + 1));
      } else {
        const element = elements[i].getElement();
        // Only translate elements that can be edited as text; skip images and
        // other non-text elements.
        if (element.editAsText) {
          const elementText = element.asText().getText();
          // This check is necessary to exclude images, which return a blank
          // text element.
          if (elementText) {
            text.push(elementText);
          }
        }
      }
    }
  }
  if (!text.length) throw new Error('Please select some text.');
  return text;
}

/**
 * Gets the stored user preferences for the origin and destination languages,
 * if they exist.
 * This method is only used by the regular add-on, and is never called by
 * the mobile add-on version.
 *
 * @return {Object} The user's origin and destination language preferences, if
 *     they exist.
 */
function getPreferences() {
  const userProperties = PropertiesService.getUserProperties();
  return {
    originLang: userProperties.getProperty('originLang'),
    destLang: userProperties.getProperty('destLang')
  };
}

/**
 * Gets the user-selected text and translates it from the origin language to the
 * destination language. The languages are notated by their two-letter short
 * form. For example, English is 'en', and Spanish is 'es'. The origin language
 * may be specified as an empty string to indicate that Google Translate should
 * auto-detect the language.
 *
 * @param {string} origin The two-letter short form for the origin language.
 * @param {string} dest The two-letter short form for the destination language.
 * @param {boolean} savePrefs Whether to save the origin and destination
 *     language preferences.
 * @return {Object} Object containing the original text and the result of the
 *     translation.
 */
function getTextAndTranslation(origin, dest, savePrefs) {
  if (savePrefs) {
    PropertiesService.getUserProperties()
        .setProperty('originLang', origin)
        .setProperty('destLang', dest);
  }
  const text = getSelectedText().join('\n');
  return {
    text: text,
    translation: translateText(text, origin, dest)
  };
}

/**
 * Replaces the text of the current selection with the provided text, or
 * inserts text at the current cursor location. (There will always be either
 * a selection or a cursor.) If multiple elements are selected, only inserts the
 * translated text in the first element that can contain text and removes the
 * other elements.
 *
 * @param {string} newText The text with which to replace the current selection.
 */
function insertText(newText) {
  const selection = DocumentApp.getActiveDocument().getSelection();
  if (selection) {
    let replaced = false;
    const elements = selection.getSelectedElements();
    if (elements.length === 1 && elements[0].getElement().getType() ===
      DocumentApp.ElementType.INLINE_IMAGE) {
      throw new Error('Can\'t insert text into an image.');
    }
    for (let i = 0; i < elements.length; ++i) {
      if (elements[i].isPartial()) {
        const element = elements[i].getElement().asText();
        const startIndex = elements[i].getStartOffset();
        const endIndex = elements[i].getEndOffsetInclusive();
        element.deleteText(startIndex, endIndex);
        if (!replaced) {
          element.insertText(startIndex, newText);
          replaced = true;
        } else {
          // This block handles a selection that ends with a partial element. We
          // want to copy this partial text to the previous element so we don't
          // have a line-break before the last partial.
          const parent = element.getParent();
          const remainingText = element.getText().substring(endIndex + 1);
          parent.getPreviousSibling().asText().appendText(remainingText);
          // We cannot remove the last paragraph of a doc. If this is the case,
          // just remove the text within the last paragraph instead.
          if (parent.getNextSibling()) {
            parent.removeFromParent();
          } else {
            element.removeFromParent();
          }
        }
      } else {
        const element = elements[i].getElement();
        if (!replaced && element.editAsText) {
          // Only translate elements that can be edited as text, removing other
          // elements.
          element.clear();
          element.asText().setText(newText);
          replaced = true;
        } else {
          // We cannot remove the last paragraph of a doc. If this is the case,
          // just clear the element.
          if (element.getNextSibling()) {
            element.removeFromParent();
          } else {
            element.clear();
          }
        }
      }
    }
  } else {
    const cursor = DocumentApp.getActiveDocument().getCursor();
    const surroundingText = cursor.getSurroundingText().getText();
    const surroundingTextOffset = cursor.getSurroundingTextOffset();

    // If the cursor follows or preceds a non-space character, insert a space
    // between the character and the translation. Otherwise, just insert the
    // translation.
    if (surroundingTextOffset > 0) {
      if (surroundingText.charAt(surroundingTextOffset - 1) !== ' ') {
        newText = ' ' + newText;
      }
    }
    if (surroundingTextOffset < surroundingText.length) {
      if (surroundingText.charAt(surroundingTextOffset) !== ' ') {
        newText += ' ';
      }
    }
    cursor.insertText(newText);
  }
}

/**
 * Given text, translate it from the origin language to the destination
 * language. The languages are notated by their two-letter short form. For
 * example, English is 'en', and Spanish is 'es'. The origin language may be
 * specified as an empty string to indicate that Google Translate should
 * auto-detect the language.
 *
 * @param {string} text text to translate.
 * @param {string} origin The two-letter short form for the origin language.
 * @param {string} dest The two-letter short form for the destination language.
 * @return {string} The result of the translation, or the original text if
 *     origin and dest languages are the same.
 */
function translateText(text, origin, dest) {
  if (origin === dest) return text;
  return LanguageApp.translate(text, origin, dest);
}
```

---

### Add-on UI
<em>Language code reference: [https://cloud.google.com/translate/docs/languages](https://cloud.google.com/translate/docs/languages)</em>

<em>sidebar.html</em>
```html
<!DOCTYPE html>
<html>
<head>
  <base target="_top">
  <link rel="stylesheet" href="https://ssl.gstatic.com/docs/script/css/add-ons1.css">
  <!-- The CSS package above applies Google styling to buttons and other elements. -->

  <style>
    .branding-below {
      bottom: 56px;
      top: 0;
    }
    .branding-text {
      left: 7px;
      position: relative;
      top: 3px;
    }
    .col-contain {
      overflow: hidden;
    }
    .col-one {
      float: left;
      width: 50%;
    }
    .logo {
      vertical-align: middle;
    }
    .radio-spacer {
      height: 20px;
    }
    .width-100 {
      width: 100%;
    }
  </style>
  <title></title>
</head>
<body>
<div class="sidebar branding-below">
  <form>
    <div class="block col-contain">
      <div class="col-one">
        <b>Selected text</b>
        <div>
          <input type="radio" name="origin" id="radio-origin-auto" value="" checked="checked">
          <label for="radio-origin-auto">Auto-detect</label>
        </div>
        <div>
          <input type="radio" name="origin" id="radio-dest-zh-TW" value="zh-TW">
          <label for="radio-dest-zh-TW">Traditional Chinese</label>
        </div>
        <div>
          <input type="radio" name="origin" id="radio-origin-en" value="en">
          <label for="radio-origin-en">English</label>
        </div>
        <div>
          <input type="radio" name="origin" id="radio-origin-fr" value="fr">
          <label for="radio-origin-fr">French</label>
        </div>
        <div>
          <input type="radio" name="origin" id="radio-origin-de" value="de">
          <label for="radio-origin-de">German</label>
        </div>
        <div>
          <input type="radio" name="origin" id="radio-origin-ja" value="ja">
          <label for="radio-origin-ja">Japanese</label>
        </div>
        <div>
          <input type="radio" name="origin" id="radio-origin-es" value="es">
          <label for="radio-origin-es">Spanish</label>
        </div>
      </div>
      <div>
        <b>Translate into</b>
        <div class="radio-spacer">
        </div>
        <div>
          <input type="radio" name="dest" id="radio-dest-zh-TW" value="zh-TW">
          <label for="radio-dest-zh-TW">Traditional Chinese</label>
        </div>
        <div>
          <input type="radio" name="dest" id="radio-dest-en" value="en">
          <label for="radio-dest-en">English</label>
        </div>
        <div>
          <input type="radio" name="dest" id="radio-dest-fr" value="fr">
          <label for="radio-dest-fr">French</label>
        </div>
        <div>
          <input type="radio" name="dest" id="radio-dest-de" value="de">
          <label for="radio-dest-de">German</label>
        </div>
        <div>
          <input type="radio" name="dest" id="radio-dest-ja" value="ja" checked="checked">
          <label for="radio-dest-ja">Japanese</label>
        </div>
        <div>
          <input type="radio" name="dest" id="radio-dest-es" value="es">
          <label for="radio-dest-es">Spanish</label>
        </div>
      </div>
    </div>
    <div class="block form-group">
      <label for="translated-text"><b>Translation</b></label>
      <textarea class="width-100" id="translated-text" rows="10"></textarea>
    </div>
    <div class="block">
      <input type="checkbox" id="save-prefs">
      <label for="save-prefs">Use these languages by default</label>
    </div>
    <div class="block" id="button-bar">
      <button class="blue" id="run-translation">Translate</button>
      <button id="insert-text">Insert</button>
    </div>
  </form>
</div>

<div class="sidebar bottom">
  <img alt="Add-on logo" class="logo" src="https://www.gstatic.com/images/branding/product/1x/translate_48dp.png" width="27" height="27">
  <span class="gray branding-text">Translate sample by Google</span>
</div>

<script src="//ajax.googleapis.com/ajax/libs/jquery/1.9.1/jquery.min.js"></script>
<script>
  /**
   * On document load, assign click handlers to each button and try to load the
   * user's origin and destination language preferences if previously set.
   */
  $(function() {
    $('#run-translation').click(runTranslation);
    $('#insert-text').click(insertText);
    google.script.run.withSuccessHandler(loadPreferences)
            .withFailureHandler(showError).getPreferences();
  });

  /**
   * Callback function that populates the origin and destination selection
   * boxes with user preferences from the server.
   *
   * @param {Object} languagePrefs The saved origin and destination languages.
   */
  function loadPreferences(languagePrefs) {
    $('input:radio[name="origin"]')
            .filter('[value=' + languagePrefs.originLang + ']')
            .attr('checked', true);
    $('input:radio[name="dest"]')
            .filter('[value=' + languagePrefs.destLang + ']')
            .attr('checked', true);
  }

  /**
   * Runs a server-side function to translate the user-selected text and update
   * the sidebar UI with the resulting translation.
   */
  function runTranslation() {
    this.disabled = true;
    $('#error').remove();
    const origin = $('input[name=origin]:checked').val();
    const dest = $('input[name=dest]:checked').val();
    const savePrefs = $('#save-prefs').is(':checked');
    google.script.run
            .withSuccessHandler(
                    function(textAndTranslation, element) {
                      $('#translated-text').val(textAndTranslation.translation);
                      element.disabled = false;
                    })
            .withFailureHandler(
                    function(msg, element) {
                      showError(msg, $('#button-bar'));
                      element.disabled = false;
                    })
            .withUserObject(this)
            .getTextAndTranslation(origin, dest, savePrefs);
  }

  /**
   * Runs a server-side function to insert the translated text into the document
   * at the user's cursor or selection.
   */
  function insertText() {
    this.disabled = true;
    $('#error').remove();
    google.script.run
            .withSuccessHandler(
                    function(returnSuccess, element) {
                      element.disabled = false;
                    })
            .withFailureHandler(
                    function(msg, element) {
                      showError(msg, $('#button-bar'));
                      element.disabled = false;
                    })
            .withUserObject(this)
            .insertText($('#translated-text').val());
  }

  /**
   * Inserts a div that contains an error message after a given element.
   *
   * @param {string} msg The error message to display.
   * @param {DOMElement} element The element after which to display the error.
   */
  function showError(msg, element) {
    const div = $('<div id="error" class="error">' + msg + '</div>');
    $(element).after(div);
  }
</script>
</body>
</html>
```
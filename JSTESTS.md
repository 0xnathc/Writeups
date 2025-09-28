# JS TESTS

API str handling

```jsx
const fs = require('fs');
const path = require('path');

/**
 * Writes a JSON string to a dynamically created new file.
 * @param {string} jsonString - The JSON string to write.
 * @param {string} [outputDir='./output'] - Directory where JSON files will be stored.
 */

function writeJsonToNewFile(jsonString, outputDir = './output') {
  try {
    // Parse JSON to validate it
    const jsonData = JSON.parse(jsonString);

    // Ensure output directory exists
    if (!fs.existsSync(outputDir)) {
      fs.mkdirSync(outputDir, { recursive: true });
    }

    // Create unique filename with timestamp
    const timestamp = Date.now();
    const filename = `file_${timestamp}.json`;
    const outputPath = path.join(outputDir, filename);

    // Write JSON to file
    fs.writeFileSync(outputPath, JSON.stringify(jsonData, null, 2), 'utf-8');
    console.log(`JSON written successfully to ${outputPath}`);
  } catch (err) {
    console.error('Invalid JSON string or file write error:', err.message);
  }
}

// Example usage:
const inputJsonString = '{"name": "Alice", "age": 30, "city": "Wonderland"}';
writeJsonToNewFile(inputJsonString);
```

multi filter option

```jsx
// Example test function (will be the mapping and string comparison/ searching stuff)
export function testFunction(input) {
  console.log("Test function received:", input);
}

import React, { useState, useEffect } from "react";
import { processStateString } from "./stringProcessor";

//set inputString 
const [inputString, setInputString] = useState("");

// Run processor whenever inputString changes
useEffect(() => {
    processStateString(inputString);
}, [inputString]);

// Function that processes a comma-separated string
function processStateString(stateString) {
  if (!stateString) return;

  // Split string into array, trimming whitespace
  const valuesArray = stateString.split(",").map(val => val.trim());

  // Iterate and call test function
  valuesArray.forEach(value => {
    testFunction(value);
  });
}

//how to call in input string
return (
    <div className="p-4">
    <h2 className="text-lg font-bold">Comma-Separated Processor</h2>
    <input
        type="text"
        value={inputString}
        onChange={e => setInputString(e.target.value)}
        className="border rounded p-2 w-full mt-2"
      />
    </div>
);
```

read in preset

```jsx
import React, { useState } from "react";
import presetData from "./presets.json"; // JSON is a top-level array

function PresetLoader() {
  const [tempPreset, setTempPreset] = useState(null); // store entire object
  const [selectedPreset, setSelectedPreset] = useState(""); // currently selected name

  /**
   * Loads the entire object from JSON based on its `name`
   * @param {string} presetName
   */
  function loadPreset(presetName) {
    if (!presetData || !Array.isArray(presetData)) {
      console.error("Preset data is missing or not an array.");
      return;
    }

    const preset = presetData.find(p => p.name === presetName);

    if (preset) {
      setTempPreset(preset);
    } else {
      console.warn(`Preset with name "${presetName}" not found.`);
      setTempPreset(null);
    }
  }

  /**
   * Handle dropdown change
   */
  const handleChange = (e) => {
    const name = e.target.value;
    setSelectedPreset(name);
    loadPreset(name);
  };

  return (
    <div className="p-4">
      <h2 className="text-lg font-bold">Preset Loader</h2>

      {/* Dropdown menu */}
      <select
        value={selectedPreset}
        onChange={handleChange}
        className="border rounded p-2 mt-2"
      >
        <option value="">-- Select a preset --</option>
        {presetData.map((preset) => (
          <option key={preset.name} value={preset.name}>
            {preset.name}
          </option>
        ))}
      </select>

      <div className="mt-4">
        <h3 className="font-semibold">tempPreset State:</h3>
        <pre className="bg-gray-100 p-2 rounded">
          {tempPreset ? JSON.stringify(tempPreset, null, 2) : "No preset loaded"}
        </pre>
      </div>
    </div>
  );
}

export default PresetLoader;
```
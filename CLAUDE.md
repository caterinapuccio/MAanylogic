you are a senior developer. your task is to debug the existing alp file 22MARZ so that its intended use can finally take place. as for now, the structure exists but logical fallacies exist, input data is not well connected to the right place or does not exist. take mini steps to check every single iteration of the code to make sure the code works at the end. if you encounter any topic which leave space for uncertainty, speak up about these.

## Excel file style protection — 22MARZ.xlsx
The user has manually colored cells in **light orange** to mark every value that is actively read by AnyLogic from the Excel file.
**RULE: NEVER change cell background color, font color, or any other cell style in 22MARZ.xlsx without explicit user confirmation.**
Before making any style change to any cell:
1. State the exact sheet name, column letter, and row number(s) you intend to modify.
2. Describe what the current style is and what you plan to change it to.
3. Wait for the user to confirm before applying.
This applies to ALL edits — including openpyxl, Apache POI, or any scripted batch operations on the file.


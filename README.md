# HDFC Infinia Statement Extractor — Claude Skill

Extracts domestic and international transactions from HDFC Infinia Credit Card
PDF statements and produces a formatted `.xlsx` workbook.

## What it does
- Reads any HDFC Infinia statement PDF
- Outputs two-sheet Excel: Domestic + International
- Matches house style: HDFC blue headers, alternating rows, CR rows in green

## Install
Download `hdfc-statement-extractor.skill` and install it in Claude.ai via
Settings → Skills → Install from file.

## Trigger phrases
- "Extract transactions from this statement"
- "Put this in a spreadsheet"
- "Same format as last time"

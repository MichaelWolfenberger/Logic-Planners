\# Logic & Formula Standards

\#\# Google Sheets / Excel Engineering Rules  
1\.  \*\*Modern Functions:\*\*   
    \*   ALWAYS use \`XLOOKUP\` over \`VLOOKUP\` or \`HLOOKUP\`.  
    \*   ALWAYS use \`IFS\` instead of nested \`IF\` statements for better readability.  
    \*   Use \`LET\` for complex formulas to declare variables and improve performance.  
2\.  \*\*Error Handling:\*\*   
    \*   Wrap all user-facing calculations in \`IFERROR(\[formula\], "")\` or \`IF(ISBLANK(\[cell\]), "", \[formula\])\` to prevent \`\#N/A\` or \`\#DIV/0\!\` errors on empty dashboards.  
3\.  \*\*Array Formulas:\*\* Prefer \`ARRAYFORMULA\` in the header row for column-wide calculations to prevent users from accidentally deleting a formula in a single cell.

\#\# Notion Database Engineering Rules  
1\.  \*\*Naming Conventions:\*\*   
    \*   Databases: PascalCase or Title Case (e.g., \`ExpenseTracker\`).  
    \*   Properties: snake\_case or lowercase with spaces (e.g., \`amount\_due\`, \`due\_date\`).  
2\.  \*\*Relations & Rollups:\*\* Minimize two-way relations if one-way solves the problem. Always hide helper/rollup properties from the main dashboard views.  
3\.  \*\*Formula Syntax:\*\* Use Notion's Formula 2.0 syntax (dot notation). Example: \`prop("Tasks").map(current.prop("Status") \== "Done").length()\`  

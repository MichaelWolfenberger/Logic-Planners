\# Template Design System & UX Rules

\#\# Visual Language (Spreadsheets & Dashboards)  
\*   \*\*Typography:\*\* Sans-serif only (e.g., Inter, Roboto, or Arial).   
\*   \*\*Color Palette (Hex Codes):\*\*  
    \*   Primary Action/Highlight: \`\#2563EB\` (Blue)  
    \*   Background (Headers): \`\#F1F5F9\` (Slate 100\)  
    \*   Text (Primary): \`\#0F172A\` (Slate 900\)  
    \*   Text (Muted/Instructions): \`\#64748B\` (Slate 500\)  
    \*   Error/Alert: \`\#EF4444\` (Red)  
    \*   Success: \`\#10B981\` (Green)

\#\# Architectural Rules  
1\.  \*\*The Setup Tab:\*\* All templates MUST include a dedicated "Setup" or "Config" tab where users input global variables (categories, names, tax rates). No hardcoding variables in formulas.  
2\.  \*\*State Separation:\*\* Visually separate "Input Cells" (where users type) from "Calculation Cells" (where formulas live). Input cells should have a subtle background color; Calculation cells should be locked/protected.  
3\.  \*\*Data Validation:\*\* Use dropdowns for categorical data to prevent user entry errors.  
4\.  \*\*Frozen Panes:\*\* Always freeze the top row containing column headers.  

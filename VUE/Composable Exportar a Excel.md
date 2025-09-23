#vue #composable #code

	
-  Exportar a Excel (general.js) usa la funcionalidad propia de JqGrid (habría que reemplazar)
```javascript title:"Exportar a excel opts" fold:true
var exportOptions = {  
    includeLabels : true,  
    includeGroupHeader : true,  
    includeFooter : false,  
    fileName : "registros-error.xlsx",  
    mimetype : "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",  
    maxlength : 40,  
    onBeforeExport : null,  
    replaceStr : null  
};
```
```javascript title:"Exportar a excel simple" fold:true  
function exportExcel(grid, nombre, orderCol) {  
    // Solo se genera el excel si existen registros  
    var records = $(grid).jqGrid('getGridParam', 'records');  
    if (records) {  
       if (nombre !== undefined) {  
          exportOptions.fileName = nombre;  
       }  
//     $(grid).jqGrid('exportToExcel', exportOptions);  
       if(orderCol) {  
          $(grid).groupingRemove();  
          $(grid).jqGrid('exportToExcel', exportOptions);  
          //$(grid).groupingGroupBy(orderCol);  
       } else{  
          $(grid).jqGrid('exportToExcel', exportOptions);  
       }  
    }  
}
```
```javascript title:"Exportar a excel con selección de columnas" fold:true
function exportExcelWithColsToShowOrHide(grid, nombre, orderCol, colsToShow, colsToHide,groupCol) {  
    // Solo se genera el excel si existen registros  
    var records = $(grid).jqGrid('getGridParam', 'records');  
    if (records) {  
       if (nombre !== undefined) {  
          exportOptions.fileName = nombre;  
       }  
//     $(grid).jqGrid('exportToExcel', exportOptions);  
       if(colsToShow){  
          for (var i=0; i<colsToShow.length; i++) {   
             $(grid).showCol(colsToShow[i]);  
          }  
       }  
       if(colsToHide){  
          for (var i=0; i<colsToHide.length; i++) {   
             $(grid).hideCol(colsToHide[i]);  
          }  
       }          
       if(groupCol){  
          $(grid).groupingRemove();  
          $(grid).jqGrid('exportToExcel', exportOptions);  
          $(grid).groupingGroupBy(groupCol);  
       }else{  
          $(grid).jqGrid('exportToExcel', exportOptions);  
       }   
       if(colsToShow){  
          for (var i=0; i<colsToShow.length; i++) {   
             $(grid).hideCol(colsToShow[i]);  
          }  
       }  
       if(colsToHide){  
          for (var i=0; i<colsToHide.length; i++) {   
             $(grid).showCol(colsToHide[i]);  
          }  
       }  
    }  
}
```

-  Capturar datos de la tabla
```html title:"en la definición del datatable" fold:true
ref="dt"
```
```javascript title:"en el script" fold:true
// importar ref
import { ref } from 'vue';
// declarar dt
const dt = ref();
// usar
dt.value
```
```javascript title:"Filtrar para incluir solo las columnas seleccionadas" fold:true
  const encabezadoFiltrado = cols.map(col => col.field).join(',')
  const filasFiltradas = datos
  .map(fila =>cols.map(col => fila[col.field]).join(','))
  .join('\n');
  const csvContent = `${encabezadoFiltrado}\n${filasFiltradas}`
```

- Nuevo plan:
	- Usar el exportCSV de PrimeVue 
	- usar una librería para convertir csv en excel
	- para ocultar columnas y/o evitar que se exporten, en la declaración de la columna:
		- agregar `hidden` para ocultarla (nada para mostrarla)
		- agregar `:exportable=false` para que no se exporte (nada para que se exporte)
		- agregar `exportHeader="Número de OT"` para ponerle un nombre a la columna en Excel diferente de el que se muestra en la tabla

-  Primer intento de composables
```javascript title:useExcelExport.js fold:true
import ExcelJS from "exceljs";

export function useExcelExport() {
  async function exportToExcel({ rows, fields, filename = "export.xlsx", columnTypes = {} }) {
    const workbook = new ExcelJS.Workbook();
    const worksheet = workbook.addWorksheet("Data");
    // Insert header row
    worksheet.addRow(fields);
    // Insert data rows
    rows.forEach(row => {
      const newRow = worksheet.addRow(row);
      row.forEach((value, colIndex) => {
        const field = fields[colIndex];
        const type = columnTypes[field] || "string";
        const cell = newRow.getCell(colIndex + 1);
        switch (type) {
          case "number":
            cell.value = isNaN(value) || value.trim() === "" ? value : Number(value);
            break;
          case "date":
            cell.value = value ? new Date(value) : null;
            break;
          default:
            cell.value = value;
        }
      });
    });
    // Download file
    const buffer = await workbook.xlsx.writeBuffer();
    const blob = new Blob([buffer], {
      type: "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"
    });
    const link = document.createElement("a");
    link.href = URL.createObjectURL(blob);
    link.download = filename;
    link.click();
    URL.revokeObjectURL(link.href);
  }
  return { exportToExcel };
}
```
```javascript title:useExcelTransform.js fold:true
export function useExcelTransform() {
  // Reorder columns based on exportOrder array
  function reorderColumns(rows, fields, exportOrder) {
    if (!exportOrder) return { rows, fields };
    const reorderedFields = exportOrder;
    const reorderedRows = rows.map(row =>
      reorderedFields.map(field => row[fields.indexOf(field)])
    );
    return { rows: reorderedRows, fields: reorderedFields };
  }

  // Group rows by a column
  function groupBy(rows, fields, groupField) {
    if (!groupField) return rows;
    const idx = fields.indexOf(groupField);
    const grouped = {};
    rows.forEach(row => {
      const key = row[idx];
      if (!grouped[key]) grouped[key] = [];
      grouped[key].push(row);
    });
    const finalRows = [];
    Object.keys(grouped).forEach(key => {
      finalRows.push([key]); // header row for the group
      finalRows.push(...grouped[key]);
    });
    return finalRows;
  }

  // Convert CSV from PrimeVue → { rows, fields }
  function parseFromDataTable(dtRef) {
    const csv = dtRef.value.exportCSV({ custom: true });
    const rows = csv.split("\n").map(r => r.split(","));
    const fields = dtRef.value.columns.map(c => c.props.field);
    // First row is header, remove it
    return { rows: rows.slice(1), fields };
  }

  return { reorderColumns, groupBy, parseFromDataTable };
}
```
		Uso en componente
```html title:template fold:true
<template>
  <DataTable ref="dt" :value="customers">
    <Column field="worker" header="Worker" />
    <Column field="workorder_number" header="Work Order" />
    <Column field="cost" header="Cost" />
    <Column field="createdAt" header="Created At" />
  </DataTable>
  <Button label="Export Custom Excel" @click="exportCustom" />
</template>
```
```javascript title:script fold:true
<script setup>
import { ref } from "vue";
import { useExcelExport } from "@/composables/useExcelExport";
import { useExcelTransform } from "@/composables/useExcelTransform";

const dt = ref(null);
const { exportToExcel } = useExcelExport();
const { parseFromDataTable, reorderColumns, groupBy } = useExcelTransform();

function exportCustom() {
  // Step 1: Parse data from PrimeVue DataTable
  let { rows, fields } = parseFromDataTable(dt);
  // Step 2: Reorder columns
  const exportOrder = ["createdAt", "workorder_number", "cost"];
  ({ rows, fields } = reorderColumns(rows, fields, exportOrder));
  // Step 3: Group by worker
  rows = groupBy(rows, fields, "worker");
  // Step 4: Export to Excel
  exportToExcel({
    rows,
    fields,
    filename: "custom.xlsx",
    columnTypes: { cost: "number", createdAt: "date" }
  });
}
</script>
```


-  Composable base funcionando
	- El planteo anterior no funcionó.  Nueva versión, tomo los datos directo de la tabla.
	- Problema: si se usan columnas dinámicas, sólo exporta lo que está mostrando.
```javascript title:useExcelExport.js fold:true
import ExcelJS from "exceljs";

export function useExcelExport() {
  
	function parseDataFromTable(dtRef) {
	    if (!dtRef.value) return { rows: [], fields: [] };
	    const exportableColumns = dtRef.value.columns.filter(
	        c => c.props.exportable !== false
	    );
	    const fields = exportableColumns.map(c => c.props.field);
	    const rows = dtRef.value.filteredValue ?? dtRef.value.value; 
	    return { rows, fields };
	}

	async function exportToExcel({ rows, fields, filename = "datos.xlsx", columnTypes = {} }) {
	    const workbook = new ExcelJS.Workbook();
	    const worksheet = workbook.addWorksheet("Datos");
	    worksheet.addRow(fields);
	        rows.forEach(row => {
	            const newRow = worksheet.addRow(
	                fields.map(field => row[field] ?? "") 
	            );
	        fields.forEach((field, colIndex) => {
	            const value = row[field];
	            const type = columnTypes[field] || "string";
	            const cell = newRow.getCell(colIndex + 1);
	            switch (type) {
	            case "number":
	                cell.value = isNaN(value) || value === "" ? null : Number(value);
	                break;
	            case "date":
	                cell.value = value ? new Date(value) : null;
	                break;
	            default:
	                cell.value = value;
	            }
	        });
	    });
	    const buffer = await workbook.xlsx.writeBuffer();
	    const blob = new Blob([buffer], {
	      type: "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"
	    });
	    const link = document.createElement("a");
	    link.href = URL.createObjectURL(blob);
	    link.download = filename;
	    link.click();
	    URL.revokeObjectURL(link.href);
	}

	function groupBy(rows, groupField) {
	  if (!groupField) return rows;
	  const grouped = {};
	  rows.forEach(row => {
	    const key = row[groupField] ?? "—"; // default key if missing
	    if (!grouped[key]) grouped[key] = [];
	    grouped[key].push(row);
	  });
	  const finalRows = [];
	  Object.keys(grouped).forEach(key => {
	    finalRows.push({ __group: key }); // header marker row
	    finalRows.push(...grouped[key]);
	  });
	  return finalRows;
	}

  return { parseDataFromTable, exportToExcel, groupBy };
}
```
	Uso
```javascript title:"composable básico sin formato columnas" fold:true
import { useExcelExport } from '@/composables/useExportExcel';

const { exportToExcel, parseDataFromTable } = useExcelExport();

// ponerle a esta función el mismo nombre que se le puso en @click
const exportarExcel = () => {
  //console.log('Datos a exportar:', dt.value.value);
  let { rows, fields } = parseDataFromTable(dt);
  exportToExcel({
    rows,
    fields,
    filename: "prueba.xlsx"
  });
};
```
```javascript title:"composable básico con formato columnas" fold:true
import { useExcelExport } from '@/composables/useExportExcel';

const { exportToExcel, parseDataFromTable } = useExcelExport();

const dt = ref();
// ponerle a esta función el mismo nombre que se le puso en @click
const exportarExcel = () => {
  //console.log('Datos a exportar:', dt.value.value);
  let { rows, fields } = parseDataFromTable(dt);
  exportToExcel({
    rows,
    fields,
    filename: "prueba.xlsx"
    // sólo se necesitan declarar las que no se exporten como string
    columnTypes: {
      cantidad: "number",
      fecha: "date"
    }
  });
};
```

- Composable completo funcionando (columnas estáticas)
```javascript title:useExcelExport.js fold:true
import ExcelJS from "exceljs";

export function useExcelExport() {

  async function exportToExcel({ rows, fields, columns, filename = "datos.xlsx", columnTypes = {}, groupField }) {
    const workbook = new ExcelJS.Workbook();
    const worksheet = workbook.addWorksheet("Datos");
    // encabezado con nombre de columna    
    const headerRow = worksheet.addRow(
      fields.map(field => {
        const colDef = columns.find(c => c.field === field);
        return colDef ? colDef.header : field;
      })
    );
    // estilo encabezado
    headerRow.eachCell(cell => {
      cell.font = { bold: true };
    });
    // datos
    if (!groupField || !fields.includes(groupField)) {
      rows.forEach(row => addRowToWorksheet(worksheet, row, fields, columnTypes));
    } else {
      const grouped = {};
      rows.forEach(row => {
        const key = row[groupField] ?? 'undefined';
        if (!grouped[key]) grouped[key] = [];
        grouped[key].push(row);
      });
      Object.keys(grouped).forEach(key => {
        const columnDef = columns.find(c => c.field === groupField);
        const groupHeaderLabel = columnDef?.header || groupField;
        const headerRow = worksheet.addRow([`${groupHeaderLabel}: ${key}`]);
        worksheet.mergeCells(headerRow.number, 1, headerRow.number, fields.length);
        headerRow.getCell(1).font = { bold: true };
        headerRow.getCell(1).alignment = { horizontal: 'left' };
        grouped[key].forEach(row => addRowToWorksheet(worksheet, row, fields, columnTypes));
      });
    }
    // escribir a un archivo y descargar
    const buffer = await workbook.xlsx.writeBuffer();
    const blob = new Blob([buffer], {
      type: "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"
    });
    const link = document.createElement("a");
    link.href = URL.createObjectURL(blob);
    link.download = filename;
    link.click();
    URL.revokeObjectURL(link.href);
  }

  function groupBy(rows, fields, groupField) {
    if (!groupField) return rows;
    const idx = fields.indexOf(groupField);
    if (idx === -1) return rows;
    const grouped = {};
    rows.forEach(row => {
      const key = row[idx] ?? 'undefined';
      if (!grouped[key]) grouped[key] = [];
      grouped[key].push(row);
    });
    const finalRows = [];
    Object.keys(grouped).forEach(key => {
      finalRows.push([{ __group: `${groupField}: ${key}` }]);
      finalRows.push(...grouped[key]);
    });

    return finalRows;
  }

  function sortRows(rows, column, order = 'asc', type = 'string') {
    const sorted = [...rows].sort((a, b) => {
      let valA = a[column];
      let valB = b[column];
      switch (type) {
        case 'number':
          valA = Number(valA) || 0;
          valB = Number(valB) || 0;
          break;
        case 'date':
          valA = parseCustomDate(valA);
          valB = parseCustomDate(valB);
          if (isNaN(valA)) valA = order === 'asc' ? Infinity : -Infinity;
          if (isNaN(valB)) valB = order === 'asc' ? Infinity : -Infinity;
          break;
        case 'string':
        default:
          valA = String(valA ?? '').toLowerCase();
          valB = String(valB ?? '').toLowerCase();
          break;
      }
      if (valA < valB) return order === 'asc' ? -1 : 1;
      if (valA > valB) return order === 'asc' ? 1 : -1;
      return 0;
	  });
    return sorted;
  }

  function parseDataFromTable(dtRef) {
    if (!dtRef.value) return { rows: [], fields: [] };
    const exportableColumns = dtRef.value.columns.filter(
        c => c.props.exportable !== false
    );
    const fields = exportableColumns.map(c => c.props.field);
    const rows = dtRef.value.filteredValue ?? dtRef.value.value; 
    return { rows, fields };
  }

  function parseCustomDate(str) {
    if (!str) return NaN;
    const [datePart, timePart] = str.split(' ');
    if (!datePart) return NaN;
    const [day, month, year] = datePart.split('/').map(Number);
    let hour = 0, minute = 0, second = 0;
    if (timePart) [hour, minute, second] = timePart.split(':').map(Number);
    return new Date(year, month - 1, day, hour, minute, second).getTime();
  }

  function addRowToWorksheet(worksheet, row, fields, columnTypes) {
    const newRow = worksheet.addRow(fields.map(f => row[f] ?? ""));
    fields.forEach((field, colIndex) => {
      const value = row[field];
      const type = columnTypes[field] || "string";
      const cell = newRow.getCell(colIndex + 1);
      switch (type) {
        case "number":
          cell.value = isNaN(value) || value === "" ? null : Number(value);
          break;
        case "date":
          cell.value = value ? new Date(value) : null;
          break;
        default:
          cell.value = value;
      }
    });
  }  

  return { parseDataFromTable, exportToExcel, groupBy, sortRows };
}
```

- Para agregar opción de formato custom para fechas y números https://chatgpt.com/c/68c97fac-0a08-8321-bb6d-bb247e7bcf86
```js title:"En el composable" fold:true
  // En algún lugar al principio
  const defaultFormats = {
    number: '#,##0.00',      // Thousands separator, 2 decimals
    date: 'mm/dd/yyyy',      // US-style date format
  }
  
  // eEn la función addRowToWorksheet
  function addRowToWorksheet(worksheet, row, fields, columnTypes = {}, formats = {}) {
    const newRow = worksheet.addRow(fields.map((f) => row[f] ?? ''))

    fields.forEach((field, colIndex) => {
      const value = row[field]
      const type = columnTypes[field] || 'string'
      const cell = newRow.getCell(colIndex + 1)

      switch (type) {
        case 'number':
          if (!isNaN(value) && value !== '') {
            cell.value = Number(value)
            cell.numFmt = formats.number || defaultFormats.number
          } else {
            cell.value = null
          }
          break

        case 'date':
          if (value) {
            cell.value = new Date(value)
            cell.numFmt = formats.date || defaultFormats.date
          } else {
            cell.value = null
          }
          break

        default:
          cell.value = value
      }
    })
  }
```
```js title:"En el componente" fold:true
// Opcional: formatos especiales
const customFormats = {
  number: '#.##0,00',      // Número con dos decimales y separador de miles
  date: 'dd-mmm-yyyy'      // Formato de la fecha en el excel
}

// Nuevo llamado al composable
const exportarExcel = () => {
  let { rows, fields } = parseDataFromTable(dt)
  exportToExcel({
    rows,
    fields,
    columns: columns.value,
    filename: 'formats.xlsx',
    columnTypes: {},
    groupField: null,
    formats = {},
  })
}
```

- Formatos comunes

| Purpose                   | Format String |
| ------------------------- | ------------- |
| Integer only              | `0`           |
| Two decimals              | `0.00`        |
| Thousands sep. + decimals | `#,##0.00`    |
| Currency (USD)            | `$#,##0.00`   |
| Currency (Euro)           | `€#,##0.00`   |
| Date (MM/DD/YYYY)         | `mm/dd/yyyy`  |
| Date (DD/MM/YYYY)         | `dd/mm/yyyy`  |
| Date with month name      | `dd-mmm-yyyy` |


To implement a reusable DataGrid component that can accept rows as props, you can define a custom DataGrid component that accepts the `rows` and `setRows` props. This way, the parent component controls the rows, allowing you to use the DataGrid in multiple places with different data.

Here’s an example:

### 1. Define the Custom DataGrid Component

This component will take in `rows`, `columns`, and `setRows` as props, making it reusable across different components.

```javascript
// MyDataGrid.js
import React from 'react';
import { DataGrid } from '@mui/x-data-grid';

const MyDataGrid = ({ rows, setRows, columns }) => {
  const handleCellEditCommit = (params) => {
    const updatedRows = rows.map((row) =>
      row.id === params.id ? { ...row, [params.field]: params.value } : row
    );
    setRows(updatedRows);
  };

  return (
    <div style={{ height: 400, width: '100%' }}>
      <DataGrid
        rows={rows}
        columns={columns}
        pageSize={5}
        checkboxSelection
        onCellEditCommit={handleCellEditCommit}
      />
    </div>
  );
};

export default MyDataGrid;
```

### 2. Use the Custom DataGrid Component in Different Parent Components

Now, you can use `MyDataGrid` in any parent component and pass in different row data and a `setRows` function for updates.

```javascript
// ParentComponent.js
import React, { useState } from 'react';
import MyDataGrid from './MyDataGrid';

const ParentComponent = () => {
  const [rows, setRows] = useState([
    { id: 1, col1: 'Hello', col2: 'World' },
    { id: 2, col1: 'DataGrid', col2: 'is cool' },
  ]);

  const columns = [
    { field: 'col1', headerName: 'Column 1', width: 150, editable: true },
    { field: 'col2', headerName: 'Column 2', width: 150, editable: true },
  ];

  const resetRows = () => {
    const newRows = [
      { id: 1, col1: 'New', col2: 'Data' },
      { id: 2, col1: 'Another', col2: 'Row' },
    ];
    setRows(newRows);
  };

  return (
    <div>
      <MyDataGrid rows={rows} setRows={setRows} columns={columns} />
      <button onClick={resetRows}>Reset Rows</button>
    </div>
  );
};

export default ParentComponent;
```

### Explanation

1. **MyDataGrid Component**:
   - Receives `rows`, `setRows`, and `columns` as props, making it flexible for different data and column configurations.
   - Uses `onCellEditCommit` to update rows via `setRows`.

2. **Parent Component**:
   - Maintains control over the row data with `useState` and a `resetRows` function to update the data.
   - Passes `rows`, `setRows`, and `columns` to the custom `MyDataGrid` component.

This approach provides flexibility, allowing each parent component to control the data shown in `MyDataGrid` and perform custom reset operations as needed.
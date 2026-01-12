document.addEventListener('DOMContentLoaded', () => {
  const itemsContainer = document.getElementById('items-container');
  const newSupplierInput = document.getElementById('new-supplier');
  const addSupplierBtn = document.getElementById('add-supplier');
  const addItemBtn = document.getElementById('add-item');
  const calculateBtn = document.getElementById('calculate');
  const saveInventoryBtn = document.getElementById('save-inventory');
  const resultDiv = document.getElementById('result');
  const exportCsvBtn = document.getElementById('export-csv');
  const exportPdfBtn = document.getElementById('export-pdf');
  let suppliers = JSON.parse(localStorage.getItem('suppliers')) || [];
  let itemsData = JSON.parse(localStorage.getItem('items')) || {};
  let itemCount = 0;

  function populateSelect(select) {
    select.innerHTML = '';
    suppliers.forEach(sup => {
      const option = document.createElement('option');
      option.value = option.textContent = sup;
      select.appendChild(option);
    });
  }

  function updateAllSelects() {
    document.querySelectorAll('.supplier-select').forEach(populateSelect);
  }

  updateAllSelects();

  addSupplierBtn.addEventListener('click', () => {
    if (newSupplierInput.style.display === 'none') {
      newSupplierInput.style.display = 'block';
      addSupplierBtn.textContent = 'Save';
    } else {
      const newSup = newSupplierInput.value.trim();
      if (newSup && !suppliers.includes(newSup)) {
        suppliers.push(newSup);
        localStorage.setItem('suppliers', JSON.stringify(suppliers));
        updateAllSelects();
      }
      newSupplierInput.value = '';
      newSupplierInput.style.display = 'none';
      addSupplierBtn.textContent = 'Add Supplier';
    }
  });

  function addItemBlock() {
    const index = itemCount++;
    const block = document.createElement('div');
    block.className = 'item-block';
    block.innerHTML = `
      <h3>Item ${index + 1}</h3>
      <label for="item-${index}-name">Item Name:</label>
      <input type="text" id="item-${index}-name" required>
      <label for="item-${index}-supplier">Supplier:</label>
      <select id="item-${index}-supplier" class="supplier-select" required></select>
      <label for="item-${index}-purchase-price">Purchase Price/Unit:</label>
      <input type="number" step="0.01" id="item-${index}-purchase-price" required>
      <label for="item-${index}-handling">Handling Costs:</label>
      <input type="number" step="0.01" id="item-${index}-handling" required>
      <label for="item-${index}-storage">Storage Costs:</label>
      <input type="number" step="0.01" id="item-${index}-storage" required>
      <label for="item-${index}-containers">Packaging Containers:</label>
      <input type="number" step="0.01" id="item-${index}-containers" required>
      <label for="item-${index}-stickers">Packaging Stickers:</label>
      <input type="number" step="0.01" id="item-${index}-stickers" required>
      <label for="item-${index}-labour">Packaging Labour:</label>
      <input type="number" step="0.01" id="item-${index}-labour" required>
      <label for="item-${index}-beginning">Beginning Inventory Units:</label>
      <input type="number" id="item-${index}-beginning" required>
      <label for="item-${index}-sales">Sales Units:</label>
      <input type="number" id="item-${index}-sales" required>
      <label for="item-${index}-end">End Inventory Units:</label>
      <input type="number" id="item-${index}-end" required>
      <label for="item-${index}-losses">Losses/Spoilage Units:</label>
      <input type="number" id="item-${index}-losses" required>
      <label for="item-${index}-overhead">Overhead Costs:</label>
      <input type="number" step="0.01" id="item-${index}-overhead" required>
    `;
    itemsContainer.appendChild(block);
    populateSelect(block.querySelector('.supplier-select'));
    const nameInput = block.querySelector(`#item-${index}-name`);
    nameInput.addEventListener('change', () => {
      const name = nameInput.value.trim();
      if (itemsData[name]) {
        block.querySelector(`#item-${index}-beginning`).value = itemsData[name].currentInventory || 0;
        block.querySelector(`#item-${index}-supplier`).value = itemsData[name].supplier || '';
        block.querySelector(`#item-${index}-purchase-price`).value = itemsData[name].purchasePrice || 0;
      }
    });
  }

  addItemBtn.addEventListener('click', addItemBlock);
  addItemBlock(); // Initial item

  calculateBtn.addEventListener('click', () => {
    const startDate = document.getElementById('start-date').value;
    const endDate = document.getElementById('end-date').value;
    if (!startDate || !endDate) {
      alert('Fill dates');
      return;
    }
    const itemBlocks = document.querySelectorAll('.item-block');
    const calcData = [];
    let totalCogs = 0;
    itemBlocks.forEach((block, index) => {
      const prefix = `item-${index}-`;
      const values = {};
      ['name', 'supplier', 'purchase-price', 'handling', 'storage', 'containers', 'stickers', 'labour', 'beginning', 'sales', 'end', 'losses', 'overhead'].forEach(field => {
        const elem = block.querySelector(`#\( {prefix} \){field}`);
        values[field] = field.includes('name') || field.includes('supplier') ? elem.value : parseFloat(elem.value);
      });
      if (Object.values(values).some(v => (typeof v === 'number' && isNaN(v)) || (typeof v === 'string' && !v))) {
        alert(`Fill all fields for item ${index + 1}`);
        return;
      }
      const purchasedUnits = values['sales'] + values['losses'] + values['end'] - values['beginning'];
      if (purchasedUnits < 0) {
        alert(`Invalid inventory for ${values['name']}`);
        return;
      }
      const purchaseCost = purchasedUnits * values['purchase-price'];
      const packagingTotal = values['containers'] + values['stickers'] + values['labour'];
      const additional = values['handling'] + values['storage'] + packagingTotal + values['overhead'];
      const beginningValue = values['beginning'] * values['purchase-price'];
      const endValue = values['end'] * values['purchase-price'];
      const cogs = beginningValue + purchaseCost + additional - endValue;
      calcData.push({ ...values, purchasedUnits, cogs });
      totalCogs += cogs;
    });
    if (calcData.length !== itemBlocks.length) return; // Error occurred
    let resultHtml = `<p>Period: ${startDate} to ${endDate}</p><table><thead><tr><th>Item</th><th>Supplier</th><th>Purchased Units</th><th>COGS</th></tr></thead><tbody>`;
    calcData.forEach(d => {
      resultHtml += `<tr><td>\( {d.name}</td><td> \){d.supplier}</td><td>${d.purchasedUnits}</td><td>\[ {d.cogs.toFixed(2)}</td></tr>`;
    });
    resultHtml += `</tbody></table><p>Total COGS: \]{totalCogs.toFixed(2)}</p>`;
    resultDiv.innerHTML = resultHtml;
    saveInventoryBtn.style.display = exportCsvBtn.style.display = exportPdfBtn.style.display = 'block';
    window.currentData = { company: 'LH Ventures', startDate, endDate, items: calcData, totalCogs };
  });

  saveInventoryBtn.addEventListener('click', () => {
    const calcData = window.currentData.items;
    calcData.forEach(d => {
      itemsData[d.name] = {
        currentInventory: d.end,
        supplier: d.supplier,
        purchasePrice: d['purchase-price']
      };
    });
    localStorage.setItem('items', JSON.stringify(itemsData));
    alert('Inventories saved');
  });

  exportCsvBtn.addEventListener('click', () => {
    const d = window.currentData;
    let csv = 'Company,Start Date,End Date,Item Name,Supplier,Purchase Price,Handling,Storage,Containers,Stickers,Labour,Beginning,Sales,End,Losses,Overhead,Purchased Units,COGS\n';
    d.items.forEach(item => {
      csv += `\( {d.company}, \){d.startDate},\( {d.endDate}, \){item.name},\( {item.supplier}, \){item['purchase-price']},\( {item.handling}, \){item.storage},\( {item.containers}, \){item.stickers},\( {item.labour}, \){item.beginning},\( {item.sales}, \){item.end},\( {item.losses}, \){item.overhead},\( {item.purchasedUnits}, \){item.cogs}\n`;
    });
    const blob = new Blob([csv], { type: 'text/csv' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url; a.download = 'cogs.csv'; a.click();
  });

  exportPdfBtn.addEventListener('click', () => {
    const { jsPDF } = window.jspdf;
    const doc = new jsPDF();
    const d = window.currentData;
    let y = 10;
    doc.text('LH Ventures COGS Report', 10, y); y += 10;
    doc.text(`Period: ${d.startDate} to ${d.endDate}`, 10, y); y += 10;
    d.items.forEach(item => {
      doc.text(`Item: ${item.name}`, 10, y); y += 10;
      doc.text(`Supplier: ${item.supplier}`, 10, y); y += 10;
      doc.text(`Purchase Price/Unit: \[ {item['purchase-price']}`, 10, y); y += 10;
      doc.text(`Handling: \]{item.handling}`, 10, y); y += 10;
      doc.text(`Storage: \[ {item.storage}`, 10, y); y += 10;
      doc.text(`Containers: \]{item.containers}`, 10, y); y += 10;
      doc.text(`Stickers: \[ {item.stickers}`, 10, y); y += 10;
      doc.text(`Labour: \]{item.labour}`, 10, y); y += 10;
      doc.text(`Beginning: ${item.beginning} units`, 10, y); y += 10;
      doc.text(`Sales: ${item.sales} units`, 10, y); y += 10;
      doc.text(`End: ${item.end} units`, 10, y); y += 10;
      doc.text(`Losses: ${item.losses} units`, 10, y); y += 10;
      doc.text(`Overhead: \[ {item.overhead}`, 10, y); y += 10;
      doc.text(`Purchased Units: ${item.purchasedUnits}`, 10, y); y += 10;
      doc.text(`COGS: \]{item.cogs.toFixed(2)}`, 10, y); y += 20;
    });
    doc.text(`Total COGS: $${d.totalCogs.toFixed(2)}`, 10, y);
    doc.save('cogs.pdf');
  });
});
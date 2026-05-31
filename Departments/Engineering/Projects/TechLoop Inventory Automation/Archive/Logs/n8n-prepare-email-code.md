> [!info] Part of [[Master Note - Inventory Automation]]

# n8n Prepare Email JavaScript Code

Copy the code below and paste it into the **JavaScript Code** box of the **Prepare Email** node in your n8n workflow.

```javascript
const items = $input.all();
const criticalCount = items.filter(i => i.json.alert_level.includes('CRITICAL')).length;
const warningCount = items.filter(i => i.json.alert_level.includes('WARNING')).length;

let html = `
<!DOCTYPE html>
<html>
<head>
<style>
  body { font-family: Arial, sans-serif; line-height: 1.6; color: #333; }
  .container { max-width: 900px; margin: 0 auto; padding: 20px; }
  .header { background: #d9534f; color: white; padding: 20px; border-radius: 5px 5px 0 0; }
  .header.warning { background: #f0ad4e; }
  .content { padding: 20px; border: 1px solid #ddd; border-top: none; }
  table { width: 100%; border-collapse: collapse; margin-top: 20px; }
  th, td { padding: 10px; border-bottom: 1px solid #ddd; text-align: left; }
  th { background: #f5f5f5; }
  .critical { color: #d9534f; font-weight: bold; }
  .warning { color: #f0ad4e; font-weight: bold; }
</style>
</head>
<body>
<div class="container">
  <div class="header ${criticalCount === 0 ? 'warning' : ''}">
    <h2>Price Watchdog Report</h2>
    <p>${criticalCount} Critical Issues | ${warningCount} Warnings</p>
  </div>
  <div class="content">
    <table>
      <thead>
        <tr>
          <th>SKU</th>
          <th>Product</th>
          <th>Price</th>
          <th>RRP</th>
          <th>Cost</th>
          <th>Margin</th>
          <th>Alert</th>
        </tr>
      </thead>
      <tbody>
`;

for (const item of items) {
  const p = item.json;
  const alertClass = p.alert_level.includes('CRITICAL') ? 'critical' : 'warning';
  html += `
    <tr>
      <td>${p.sku}</td>
      <td><a href="https://techloop.com.au/wp-admin/post.php?post=${p.wc_id}&action=edit">${p.name}</a></td>
      <td>$${p.current_price}</td>
      <td>$${p.leader_rrp}</td>
      <td>$${p.cost}</td>
      <td class="${alertClass}">${p.margin_percent}%</td>
      <td class="${alertClass}">${p.alert_level}</td>
    </tr>
  `;
}

html += `
      </tbody>
    </table>
  </div>
</div>
</body>
</html>`;

return [{
  json: {
    subject: `[TechLoop Inventory] Price Alert: ${criticalCount} Critical, ${warningCount} Warnings`,
    html: html
  }
}];
```

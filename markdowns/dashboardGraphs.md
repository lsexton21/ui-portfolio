## Data Visualization - Dashboard Graphs

These are a few examples of the dozen or so graphs that I created for clients to visualize their traffic and costs.  I used chartJS as a library for the base, and added plenty of complex customization along the way.  These charts are also more impressive to see in person because you are missing the UX interactions with each graph (onClick changes, tooltips, etc.).

**See below for code**

![data graphs](/assets/dashboardGraphs.png)


---


#### As you would expect, there are so many layers within this dashboard page, but I focused on showing my code for creating the Tenant chart (which is the one on the top right)

```
import Helpers from "../../helpers/Helpers.js";
import ChartBase from "./ChartBase.js";
import ObjectHelpers from "../../helpers/ObjectHelpers.js";

class TenantsCostChart extends ChartBase {
  static _defaultOptions = {
    chartType: "doughnut",
    datalabels: true,
    label: (stat) => stat.tenant.name,
  };

  // can create as many datasets as you want,
  //but they all need to be returned in a single array
  _createDatasets() {
    this._costDataset = {
      label: "Highest Cost",
      data: [],
      backgroundColor: [
        "rgba(255, 99, 132, 0.8)",
        "rgba(255, 159, 64, 0.8)",
        "rgba(255, 205, 86, 0.8)",
        "rgba(75, 192, 192, 0.8)",
        "rgba(255, 153, 0, 0.8)",
        "rgba(54, 162, 235, 0.8)",
        "rgba(153, 102, 255, 0.8)",
        "rgba(72, 0, 255, 0.8)",
        "rgba(255, 200, 221, 0.8)",
        "rgba(0, 128, 128, 0.8)",
        "rgba(128, 0, 0, 0.8)",
        "rgba(0, 247, 255, 0.8)",
        "rgba(201, 203, 207, 0.8)",
      ],
      borderColor: ["#FFF"],
      borderWidth: 2,
    };

    return [this._costDataset];
  }

  _getTooltipOptions() {
    return {
      callbacks: {
        title: (tooltipItem) => {
          return tooltipItem[0].label;
        },
        label: (tooltipItem) => {
          return "Total Cost: " + Helpers.formatUSD(tooltipItem.raw);
        },
        footer: (tooltipItem, data) => {
          if (tooltipItem[0].label === "Other") {
            const otherObject = this._stats.slice(-1);
            let footerList = [];

            for (let org of otherObject[0].orgsList) {
              footerList.push(`- ${org.name}: ${Helpers.formatUSD(org.value)}`);
            }
            if (footerList.length > 5) {
              footerList = footerList.slice(1, 6);
              footerList.push("*For more tenants, see tenants table");
            }
            return footerList;
          }
        },
      },
    };
  }

  _getCustomPlugins() {
    const totalPlugin = {
      id: "totalPlugin",
      afterDatasetsDraw: (chart) => {
        const { ctx, $datalabels, chartArea } = chart;

        let totalCost = 0;

        for (let cost of $datalabels._labels) {
          totalCost += cost._el.$context.raw;
        }

        const responsiveFontSize = () => {
          if (chartArea.width < 300 && chartArea.width > 150) {
            return chartArea.width * 0.06;
          } else if (chartArea.width < 150) {
            return 0;
          } else {
            return 17;
          }
        };
        ctx.textAlign = "center";
        ctx.fillStyle = "darkgray";
        ctx.font = `${responsiveFontSize()}px Arial`;

        let title;

        if (this.getChartId() === "total") {
          title = "Total Cost";
        } else if (this.getChartId() === "totalInboundUsageCost") {
          title = "Inbound Cost";
        } else {
          title = "Outbound Cost";
        }

        ctx.fillText(title, chartArea.width / 2, chartArea.height / 2 - 20);
        ctx.fillText(
          "Among Tenants:",
          chartArea.width / 2,
          chartArea.height / 2
        );
        ctx.fillText(
          Helpers.formatUSD(totalCost),
          chartArea.width / 2,
          chartArea.height / 2 + 30
        );
      },
    };
    return [totalPlugin];
  }

  _getLegendOptions() {
    return {
      position: "right",
    };
  }

  _getScalesOptions() {
    return {};
  }

  _getDatalabelOptions() {
    return {
      font: {
        size: "12px",
      },
      display: false,
      formatter: function (value) {
        return `$${value}`;
      },
    };
  }

  _orderStatementsByValue(tenantStatements, path) {
    // const filteredTenants = _.filter(
    //   tenantStatements,
    //   (tenant) => ObjectHelpers.getValueByPath(tenant, path) > 0
    // );
    const sortedTenants = _.sortBy(tenantStatements, (tenant) =>
      ObjectHelpers.getValueByPath(tenant, path)
    ).reverse();
    return sortedTenants;
  }

  _addOtherLabel(statements, path) {
    if (statements.length > 12) {
      function findIndexToSlice(statements) {
        let initialTotal = 0;
        for (let statement of statements) {
          initialTotal += ObjectHelpers.getValueByPath(statement, path);
        }

        for (let i = 0; i < statements.length; i++) {
          if (i < 12) {
            if (
              ObjectHelpers.getValueByPath(statements[i], path) <
              initialTotal * 0.03
            ) {
              return i;
            }
          } else {
            return 12;
          }
        }
      }

      const indexToSlice = findIndexToSlice(statements);

      const otherStatements = statements.slice(indexToSlice);

      let sumOfOtherStatements = 0;
      let orgsList = [];
      for (let statement of otherStatements) {
        orgsList.push({
          name: statement.tenant.name,
          value: ObjectHelpers.getValueByPath(statement, path),
        });
        sumOfOtherStatements += ObjectHelpers.getValueByPath(statement, path);
      }
      statements = statements.slice(0, indexToSlice);

      if (sumOfOtherStatements > 0) {
        let totals;
        if (path === "totals.total") {
          totals = { total: sumOfOtherStatements };
        }
        if (path === "totals.inbound.conversational.cost") {
          totals = {
            inbound: {
              conversational: { cost: sumOfOtherStatements },
            },
          };
        }
        if (path === "totals.outbound.conversational.cost") {
          totals = {
            outbound: {
              conversational: { cost: sumOfOtherStatements },
            },
          };
        }
        statements.push({
          tenant: { name: "Other" },
          totals,
          orgsList,
        });
      }
    }
    return statements;
  }

  _updateRawStats(statements, chartId) {
    if (chartId === "total") {
      const sortedStatements = this._orderStatementsByValue(
        statements,
        "totals.total"
      );

      return this._addOtherLabel(sortedStatements, "totals.total");
    }

    if (chartId === "totalInboundUsageCost") {
      const sortedStatements = this._orderStatementsByValue(
        statements,
        "totals.inbound.conversational.cost"
      );

      return this._addOtherLabel(
        sortedStatements,
        "totals.inbound.conversational.cost"
      );
    }

    if (chartId === "totalOutboundUsageCost") {
      const sortedStatements = this._orderStatementsByValue(
        statements,
        "totals.outbound.conversational.cost"
      );

      return this._addOtherLabel(
        sortedStatements,
        "totals.outbound.conversational.cost"
      );
    }
  }

  // Update using same index order in createDatasets
  _updateDatasets(tenants, chartId) {
    if (chartId === "total") {
      this._costDataset.data = _.map(tenants, (tenant) => tenant.totals.total);
    }

    if (chartId === "totalInboundUsageCost") {
      this._costDataset.data = _.map(
        tenants,
        (tenant) => tenant.totals.inbound.conversational.cost
      );
    }

    if (chartId === "totalOutboundUsageCost") {
      this._costDataset.data = _.map(
        tenants,
        (tenant) => tenant.totals.outbound.conversational.cost
      );
    }
  }
}

export default TenantsCostChart;

```

[Return to main menu](../README.md)
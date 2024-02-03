# jsPDF Example

This is an example of when I needed to update our billing statement using jsPDF.  The original did not have any styling, nor did it have any Skyetel or customer information, if you can believe that.  Below is just one of three class layers that I updated to stylize the pdf document.

![jsPDF for billing statements](/assets/jsPDFStatement.png)

```
import Helpers from '../helpers/Helpers.js';
import PDFDocument from './PDFDocument.js';
import MomentFormatter from '../helpers/MomentFormatter.js';
import ClassHelpers from '../helpers/ClassHelpers.js';
import StatementBasePDFDocument from "./StatementBasePDFDocument";
import { toYesNoText } from "../helpers/BooleanHelpers";

class StatementPDFDocument extends StatementBasePDFDocument {
    constructor(options) {
        super(_.extend({
              brand: 'Skyetel'
            , totals: options.statement.totals
            , inventoryTitle: 'Transactions & Inventory Items'
            , faxTitle: 'SkyeFax'
        }, options));
    }

    _init(options) {
        super._init(options);

        this._statement = options.statement;
    }

    _print() {
        const statement = this._statement;

        const smallPadding = this._smallPadding;
        const largePadding = this._largePadding;

        this._printSkyetelHeader();

        this._addPadding(40)

        this._printStatementHeader();

        this._addPadding(20);

        this._printCustomerHeader();

        this._addPadding(15);

        this._printSpacer(80, this._y, this._pageWidth - 160, 1);

        this._addPadding(40);

        if (!statement.totals)
            this._addPadding(largePadding + smallPadding + 2);
        else {
            this._addPadding(smallPadding);

            this._printTotalsTables();

            //this._printSpacer(this._margin, this._y - 10, this._pageWidth - this._doubleMargin, 1);

            this._addPadding(40);
        }

        this._printTransactionsTable();

        this._addPadding(largePadding);

        if (!this._hasTaxes())
            this._printTaxNotice();
        else {
            //this._printSpacer(this._margin, this._y - 10, this._pageWidth - this._doubleMargin, 1);

            this._addPadding(40);

            this._printTaxesTable();
        }

        this._printStatementFooters()
    }

    _printChargesTable({ header, columns, rows, rightAlignHeaderColumnIndex, columnStyles, emptyText }) {
        const margin = this._margin;

        this._addPageIfNecessary(100);

        this._printTableHeader(header);

        this._addPadding(15);

        const rightAlignHeaderCell = StatementBasePDFDocument._rightAlignCell(rightAlignHeaderColumnIndex);

        this._doc.autoTable({
              startY: this._y
            , margin: {
                  top: margin
                , right: margin
                , bottom: margin + this._footerHeight + 10
                , left: margin
            }
            , headStyles: {
                  fillColor: 230
                , textColor: 40
            }
            , bodyStyles: {
                textColor: 0
            }
            , columns
            , body: rows
            , rowPageBreak: 'avoid'
            , columnStyles
            , didParseCell: ({ cell, column, section }) => {
                if (section === 'head')
                    rightAlignHeaderCell(cell, column);
            }
        });

        this._y = this._doc.autoTable.previous.finalY;

        if (!rows.length)
            this._printEmptyTableNote(emptyText);
    }

    _printTransactionsTable() {
        const columns = [
              { header: 'Date', dataKey: 'date' }
            , { header: 'Description', dataKey: 'description' }
            , { header: 'Amount', dataKey: 'amount' }
        ];

        const rows = _.map(this._statement.transactions, ({ date, description, amount, isDebit }) => {
            return {
                  date: MomentFormatter.format(date)
                , description
                , amount: (amount === 0 ? '' : (isDebit ? '- ' : '+ ')) + Helpers.formatUSD(amount)
            };
        });

        const columnStyles = {
            amount: {
                halign: 'right'
            }
            , description: {
                  cellWidth: 340
                , overflow: 'linebreak'
            }
        };

        this._printChargesTable({
              header: 'Transactions'
            , columns
            , rows
            , rightAlignHeaderColumnIndex: 2
            , columnStyles
            , emptyText: 'No transactions this month'
        });
    }

    _hasTaxes() {
        return this._statement.taxes;
    }

    _printTaxesTable() {
        const { taxes } = this._statement;

        const columns = [
              { header: 'Taxing Authority', dataKey: 'authority' }
            , { header: 'Taxing Description', dataKey: 'description' }
            , { header: 'Taxes Owed', dataKey: 'amount' }
            , { header: 'Exempt', dataKey: 'isExempt' }
        ];

        const rows = _.map(taxes, ({ authority, description, amount, isExempt }) => {
            return {
                  authority
                , description
                , amount: Helpers.formatUSD(amount)
                , isExempt: toYesNoText(isExempt)
            };
        });

        const columnStyles = {
            authority: {
                overflow: 'linebreak'
            }
            , description: {
                overflow: 'linebreak'
            }
            , amount: {
                minCellWidth: 70
            }
            , isExempt: {
                halign: 'right'
            }
        };

        this._printChargesTable({
              header: 'Taxes'
            , columns
            , rows
            , rightAlignHeaderColumnIndex: 3
            , columnStyles
            , emptyText: 'No tax records'
        });
    }
    
    _printEmptyTableNote(note) {
        const smallPadding = this._smallPadding;
        const doc = this._doc;
        const fontSize = 10;

        doc.setFontSize(fontSize);
        doc.setTextColor(50);

        this._addPadding(smallPadding + 6);
        this._printTextCenter(note, this._y);
        this._y += fontSize + smallPadding / 2;
    }
}

StatementPDFDocument._tableDivider = {};

ClassHelpers.copyInheritedStaticProperties(StatementPDFDocument, PDFDocument);

export default StatementPDFDocument;
```



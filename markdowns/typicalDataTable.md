## A Typical Data Table

This is just one of many examples of an interactive data table that I built for our admin and/or clients to set up their telecom network configurations.  Like this one, most tables will have dynamic values (such as counts, colors, status bars and tooltips) that change when configuration is altered.  Furthermore, each table will also have multiple buttons that open up other modals for advanced configuration.  This page alone uses about 25 ***different*** class types to come together to make this table.

**See below for code**

![Main Table](/assets/vendorTrunkGroups.png)
![Example of editable side table; accessed when clicking the "+" icon on the main table](/assets/vtgIPs.png)
![Modal that will edit the main settings of this particular table (accessed when clicking the "gear" icon on the main table.](/assets/editorModal.png)



---


#### This code example is honestly one that would be much better in person, since there are so many classes involved, but here is a snippet of the overarching table class which extends a complex ServerSideTable class that manages filters and pagination, and ties in a few modals.

```
import Table from "../Table.js";
import ServerSideTable from "../ServerSideTable.js";
import OrderableColumns from "../OrderableColumns.js";
import DataObjectsCache from "../../data/DataObjectsCache.js";
import createSearchTableFilters from "../createSearchTableFilters.js";
import DataObjectsFilter from "../../data/DataObjectsFilter.js";
import TerminationShowVendorIPAddressesModal from "../../modals/termination/TerminationShowVendorIPAddressesModal.js";
import TableRow from "../TableRow.js";
import App from "../../App.js";
import TerminationVendorTrunkGroup from "../../data/termination/TerminationVendorTrunkGroup.js";
import TerminationShowVendorsModal from "../../modals/termination/TerminationShowVendorsModal.js";
import TableHelpers from "../../helpers/TableHelpers.js";
import TerminationTableHelpers from "../../helpers/TerminationTableHelpers.js";

class TerminationVendorTrunkGroupsTable extends ServerSideTable {
  constructor($table, options) {
    super(
      $table,
      _.extend(
        {
          itemName: "Vendor Trunk Group",
          keepItemNameCase: true,
          columns: [
            Table.createColumn(
              "status",
              { width: "50px" },
              TerminationTableHelpers.createStateColumn
            ),
            Table.createColumn("name"),
            Table.createColumn("id"),
            Table.createColumn("vendor", TableHelpers.renderDataObject),
            Table.createColumn(
              "trafficClasses",
              {
                sortable: false,
              },
              TerminationVendorTrunkGroupsTable.renderTrafficClasses()
            ),
            Table.createColumn(
              "cpsLimit",
              { sortable: false },
              TerminationTableHelpers.renderCPSUtilization()
            ),

            Table.createColumn(
              "ipAddresses",
              { sortable: false },
              TerminationTableHelpers.createIPsColumn()
            ),
            Table.createColumn("notes", {
              sortable: false,
              className: "termination-notes",
            }),
          ],
          orderableColumns: new OrderableColumns([
            { index: 0, name: "status.name" },
            { index: 1, name: "name" },
            { index: 2, name: "id" },
            { index: 3, name: "vendor.name" },
          ]),
          dataObjectClass: TerminationVendorTrunkGroup,
          hasAddButton: App.accountIsAdmin(),
          hasDeleteRowButton: App.accountIsAdmin(),
          responsiveMaxWidth: 850,
          activeRowsSupported: true,
          order: [[1, "asc"]],
          headerButtons: {
            title: "Add Vendor",
            iconClass: "fa fa-plus",
            clicked: () => {
              this.vendorModal._show();
            },
          },
        },
        options
      ),
      TerminationVendorTrunkGroupsTable._searchFilters,
      DataObjectsCache.create(TerminationVendorTrunkGroup, DataObjectsFilter)
    );

    this.vendorAddressModal = new TerminationShowVendorIPAddressesModal();
    this.vendorModal = new TerminationShowVendorsModal();

    this._tableClass = "trunk-group-table";
    $table.addClass(this._tableClass);

    this["on"](
      "action.edit-ip-addresses",
      TerminationVendorTrunkGroupsTable.editVendorAddresses.bind(this)
    );

    this.vendorModal.on("rowAdded", (vendor) => {
      this._editorModal.addVendor(vendor);
    });

    this.vendorModal.on("rowRemoved", (vendor) => {
      this._editorModal.removeVendor(vendor);
    });

    this.vendorModal.on("rowUpdated", (vendor) => {
      this._editorModal.updateVendor(vendor);
    });

    TerminationTableHelpers.initStatusTooltips(this._$table);
  }

  _createNewRowDataObject(data) {
    var vendorTrunkGroup = new TerminationVendorTrunkGroup();

    for (const property in data) {
      vendorTrunkGroup[property] = data[property];
    }

    return vendorTrunkGroup;
  }

  static renderTrafficClasses() {
    return {
      type: "html",
      className: "termination-badge-column",
      render: (trafficClasses, type) => {
        if (type !== "display") {
          return "";
        }

        let trafficClassNames = [];

        for (let trafficClass of trafficClasses) {
          trafficClassNames.push(trafficClass.getName());
        }

        if (trafficClasses.length === 1) {
          return (
            `<div title="${trafficClassNames[0]}" class="termination-badge-container">` +
            `<div class="badge-number traffic-classes">${trafficClasses.length}</div>Traffic Class</div>`
          );
        } else {
          return (
            `<div title="${trafficClassNames.join(
              ", "
            )}" class="termination-badge-container">` +
            `<div class="badge-number traffic-classes">${trafficClasses.length}</div>Traffic Classes</div>`
          );
        }
      },
    };
  }

  static editVendorAddresses(tableRow) {
    const vendorTrunkGroup = tableRow.getDataObject();

    if (vendorTrunkGroup.isArchived) return;

    TableRow._highlightRowsForModal(tableRow, this.vendorAddressModal);

    this.vendorAddressModal._show(vendorTrunkGroup, tableRow);
    this.vendorAddressModal.update(vendorTrunkGroup);
  }

  update(statuses, vendors) {
    this.vendorAddressModal.updateStatuses(statuses);
    this.vendorModal.update(vendors);
  }
}

TerminationVendorTrunkGroupsTable._searchFilters = createSearchTableFilters(
  (search) => new DataObjectsFilter({ search: search })
);

export default TerminationVendorTrunkGroupsTable;

```


---
#### This is just one of four data class objects that are factored into this page.  The class that this code extends,  the "DataObject" class, is what houses the methods for each data object to use my "Request" object helper which makes CRUD requests to the API.
```
import DataObject from "../DataObject";
import TerminationTrafficClass from "./TerminationTrafficClass";
import TerminationVendorIPAddress from "./TerminationVendorIPAddress";
import RawDataConverter from "../RawDataConverter";
import TerminationVendor from "./TerminationVendor";
import TerminationTrunkGroupStatus from "./TerminationTrunkGroupStatus";
import TerminationVendorLoadBalancer from "./TerminationVendorLoadBalancer";

class TerminationVendorTrunkGroup extends DataObject {
  _setPropertiesFromRawData(rawData) {
    this.id = rawData.id;
    this.name = rawData.name;
    this.vendor = new TerminationVendor({
      id: rawData.vendor_id,
      name: rawData.vendor_name,
    });
    this.vendorLoadBalancer = rawData.vlb_id ? new TerminationVendorLoadBalancer({id: rawData.vlb_id, name: rawData.vlb_name}): null;
    this.vendorUniqueIdentifier = rawData.vendor_unique_identifier;
    this.vendorRoundingDigits = rawData.vendor_rounding_digits
    this.status = rawData.status
      ? new TerminationTrunkGroupStatus(rawData.status)
      : null;
    this.trafficClasses = rawData.traffic_classes
      ? _.map(
          rawData.traffic_classes,
          (trafficClass) => new TerminationTrafficClass(trafficClass)
        )
      : [];
    this.trafficClassIds = rawData.traffic_class_ids;
    this.cpsLimit = rawData.cps_limit;
    this.asrLimit = rawData.asr_limit
      ? this.convertDecimalToPercent(rawData.asr_limit)
      : null;
    this.ccrLimit = rawData.ccr_limit
      ? this.convertDecimalToPercent(rawData.ccr_limit)
      : null;
    this.alocLimit = rawData.aloc_limit;
    this.sdLimit = rawData.sd_limit
      ? this.convertDecimalToPercent(rawData.sd_limit)
      : null;
    this.sdLength = rawData.sd_length;
    this.billingIncrementsInitial = rawData.billing_increments_initial;
    this.billingIncrementsContinuous = rawData.billing_increments_continuous;
    this.loadDistributionAlgorithm = RawDataConverter.toEnum(
      RawDataConverter.toNumberOrNull(rawData.load_distribution_algorithm),
      TerminationVendorTrunkGroup.loadDistributionAlgorithms
    );
    this.trust404 = rawData.trust_404;
    this.createdBy = rawData.created_by;
    this.createdOn = rawData.created_on ? rawData.created_on.date : null;
    this.updatedBy = rawData.updated_by;
    this.updatedOn = rawData.updated_on ? rawData.updated_on.date : null;
    this.ipAddresses = rawData.ip_addresses
      ? _.map(
          rawData.ip_addresses,
          (ipAddress) => new TerminationVendorIPAddress(ipAddress)
        )
      : [];

    this.notes = rawData.notes;
    this.active = rawData.status
      ? rawData.status.name !== "Inactive"
        ? true
        : false
      : true;
  }

  getName() {
    return this.name;
  }

  getId() {
    return this.id;
  }

  getTrafficClassIds() {
    return this.trafficClassIds;
  }

  convertDecimalToPercent(decimal) {
    return Math.round(decimal * 100);
  }

  updateVendor(data) {
    this.vendor = new TerminationVendor(data);
  }
  
  getVLBId() {
    if (this.vendorLoadBalancer) {
      return this.vendorLoadBalancer.getId();
    } else {
      return null;
    }
  }

  _getAddRawData() {
    var rawData = {
      name: this.name,
      vendor_id: this.vendor.getId(),
      vlb_id: this.vendorLoadBalancer ? this.vendorLoadBalancer.getId(): null,
      vendor_unique_identifier: this.vendorUniqueIdentifier,
      vendor_rounding_digits: this.vendorRoundingDigits,
      status_id: this.status ? this.status.getId() : null,
      cps_limit: this.cpsLimit,
      notes: this.notes,
      asr_limit: this.asrLimit / 100,
      ccr_limit: this.ccrLimit / 100,
      aloc_limit: this.alocLimit,
      sd_limit: this.sdLimit / 100,
      sd_length: this.sdLength,
      load_distribution_algorithm: this.loadDistributionAlgorithm.rawValue,
      billing_increments_initial: this.billingIncrementsInitial,
      billing_increments_continuous: this.billingIncrementsContinuous,
      trust_404: this.trust404,
    };

    const trafficClassIDs = _.map(this.trafficClasses, (trafficClass) =>
      trafficClass.getId()
    );

    rawData.traffic_class_ids = `[${trafficClassIDs.toString()}]`;

    return rawData;
  }

  _getUpdateRawData() {
    var rawData = {};

    this._addRawDataNullableTextPropertyIfChanged(rawData, "name", "name");

    const trafficClassIDs = _.map(this.trafficClasses, (trafficClass) =>
      trafficClass.getId()
    );

    if (
      `[${trafficClassIDs.toString()}]` !==
      `[${this._initialRawData["traffic_class_ids"].toString()}]`
    ) {
      rawData.traffic_class_ids = `[${trafficClassIDs.toString()}]`;
    }

    if (this.vendor.getId() !== this._initialRawData.vendor_id) {
      rawData.vendor_id = this.vendor.getId();
    }

    if (this.vendorLoadBalancer) {
      if (this.vendorLoadBalancer.getId() !== this._initialRawData.vlb_id) {
        rawData.vlb_id = this.vendorLoadBalancer.getId();
      }
    } else {
      if (this._initialRawData.vlb_id !== null) {
        rawData.vlb_id = null;
      }
    }
    
    this._addRawDataNullableTextPropertyIfChanged(rawData, "notes", "notes");
    this._addRawDataEnumPropertyIfChanged(
      rawData,
      "loadDistributionAlgorithm",
      "load_distribution_algorithm"
    );
    this._addRawDataObjectReferenceIfChanged(
      rawData,
      "status",
      "status_id",
      "status"
    );

    if (
      this.asrLimit / 100 !==
      RawDataConverter.toNumber(this._initialRawData.asr_limit)
    ) {
      rawData.asr_limit = this.asrLimit / 100;
    }

    if (
      this.ccrLimit / 100 !==
      RawDataConverter.toNumber(this._initialRawData.ccr_limit)
    ) {
      rawData.ccr_limit = this.ccrLimit / 100;
    }

    if (
      this.sdLimit / 100 !==
      RawDataConverter.toNumber(this._initialRawData.sd_limit)
    ) {
      rawData.sd_limit = this.sdLimit / 100;
    }

    this._addRawDataNumberPropertyIfChanged(rawData, "alocLimit", "aloc_limit");
    this._addRawDataNumberPropertyIfChanged(rawData, "sdLength", "sd_length");
    this._addRawDataNumberPropertyIfChanged(rawData, "cpsLimit", "cps_limit");
    this._addRawDataNumberPropertyIfChanged(
      rawData,
      "vendorUniqueIdentifier",
      "vendor_unique_identifier"
    );
    this._addRawDataNumberPropertyIfChanged(
      rawData,
      "vendorRoundingDigits",
      "vendor_rounding_digits"
    );
    this._addRawDataNumberPropertyIfChanged(
      rawData,
      "billingIncrementsInitial",
      "billing_increments_initial"
    );
    this._addRawDataNumberPropertyIfChanged(
      rawData,
      "billingIncrementsContinuous",
      "billing_increments_continuous"
    );

    this._addRawDataNullableBooleanPropertyIfChanged(
      rawData,
      "trust404",
      "trust_404"
    );
    return rawData;
  }

  addIPAddress(ipDataObject) {
    this.ipAddresses.push(ipDataObject);
  }

  removeIPAddress(ipDataObject) {
    let newIPList = [];
    for (let ip of this.ipAddresses) {
      if (ip.getId() !== ipDataObject.getId()) {
        newIPList.push(ip);
      }
    }
    this.ipAddresses = newIPList;
  }

  updateIPAddress(ipDataObject) {
    this.removeIPAddress(ipDataObject);
    this.addIPAddress(ipDataObject);
  }

  async commitRemove(auth) {
    const originalIpAddressesLength = this.ipAddresses.length;
    let ipAddressesComplete = [];
    try {
      for (let ipAddress of this.ipAddresses) {
        ipAddressesComplete.push(await ipAddress.commitRemove(auth));
      }
    } catch (error) {
      if (isDevelopment) {
        console.log(error);
      }
      throw {
        message:
          "There was a problem deleting your ipAddresses.  Please delete them first, then try deleting the vendor trunk group again.",
      };
    }

    return super.commitRemove(auth);
  }
}
TerminationVendorTrunkGroup._defaultRawData = {
  id: null,
  name: null,
  vendor_id: null,
  vendor_name: null,
  vlb_id: null,
  vlb_name: null,
  vendor_unique_identifier: null,
  vendor_rounding_digits: 6,
  status: null,
  traffic_class: null,
  cps_limit: 5,
  asr_limit: null,
  ccr_limit: null,
  aloc_limit: null,
  sd_limit: null,
  sd_length: null,
  billing_increments_continuous: 6,
  billing_increments_initial: 6,
  load_distribution_algorithm: null,
  created_by: null,
  created_on: null,
  updated_by: null,
  updated_on: null,
  ip_addresses: [],
  notes: null,
};
TerminationVendorTrunkGroup._requestUrl = "stonehenge/vendortrunkgroups";

TerminationVendorTrunkGroup.loadDistributionAlgorithms = {
  roundRobin: {
    displayName: "Round Robin",
    rawValue: 4,
  },
  randomDestination: {
    displayName: "Random Destination",
    rawValue: 6,
  },
  priority: {
    displayName: "In order by Priority",
    rawValue: 8,
  },
};

export default TerminationVendorTrunkGroup;
```


---
#### And here is an example of the code behind the main editor modal for this table (well, one class of three because it is an extension of an extension).  This contains all the settings for setting up the editor form's inputs which creates bootstrap text and select inputs behind the scenes.
```
import EditorModal from "../EditorModal.js";
import TabbedObjectEditor from "../../components/TabbedObjectEditor.js";
import DataObjectSelectOption from "../../components/DataObjectSelectOption.js";
import TextPropertyEditor from "../../components/TextPropertyEditor.js";
import SelectPropertyEditor from "../../components/SelectPropertyEditor.js";
import ValueSelectOption from "../../components/ValueSelectOption.js";
import PropertyEditorBuilder from "../../builders/PropertyEditorBuilder.js";
import TerminationVendorTrunkGroup from "../../data/termination/TerminationVendorTrunkGroup.js";
import Helpers from "../../helpers/Helpers.js";
import TerminationTrunkGroupStatus from "../../data/termination/TerminationTrunkGroupStatus.js";
import TerminationTableHelpers from "../../helpers/TerminationTableHelpers.js";

class TerminationVendorTrunkGroupEditorModal extends EditorModal {
  constructor() {
    super({
      dataTitle: "Vendor Trunk Group",
      dialogClass: "vendor-trunk-groups",
    });

    const objectEditor = new TabbedObjectEditor([
      {
        title: "General",
        propertyEditors: this._initGeneralEditors(),
        height: 600,
      },
      {
        title: "Billing Rules",
        propertyEditors: this._initBillingRulesEditors(),
        height: 155,
      },
      {
        title: "Limits",
        propertyEditors: this._initLimitsEditors(),
        height: 350,
      },
    ]);
    objectEditor.exactLabelColumns = 4;
    this._setObjectEditor(objectEditor);
  }

  _initGeneralEditors() {
    return [
      (this.nameEditor = this._createNameEditor()),
      (this.statusEditor = this._createStatusEditor()),
      (this.loadDistributionAlgorithm =
        this._createLoadDistributionAlgorithmEditor()),
      (this.vendorUniqueIdentifierEditor =
        this._createVendorUniqueIdentifierEditor()),
      (this.vendorRoundingDigitsEditor =
        this._createVendorRoundingDigitsEditor()),
      (this.vendorEditor = this._createVendorEditor()),
      (this.vendorLoadBalancerEditor = this._createVendorLoadBalancerEditor()),
      (this.cpsLimitEditor = this._createCpsLimitEditor()),
      (this.trust404Editor = this._createBooleanEditor(
        "Trust 404",
        "trust404"
      )),
      (this.notesEditor = this._createNotesEditor()),
    ];
  }

  _initBillingRulesEditors() {
    return [
      (this.billingIncrementsInitialEditor =
        this._createBillingIncrementsInitialEditor()),
      (this.billingIncrementsContinuousEditor =
        this._createBillingIncrementsContinuousEditor()),
    ];
  }

  _initLimitsEditors() {
    return [
      (this.trafficClassesEditor = this._createTrafficClassesEditor()),
      (this.asrLimitEditor = this._createPercentEditor(
        "ASR Limit",
        "asrLimit",
        "ASR limit percent value",
        "Enter a percent value",
        1,
        100,
        {
          validateNotEmpty: true,
          emptyMessage: "Please enter an ASR limit",
          allowFloat: true,
        }
      )),
      (this.ccrLimitEditor = this._createPercentEditor(
        "CCR Limit",
        "ccrLimit",
        "CCR limit percent value",
        "Enter a percent value",
        1,
        100,
        {
          validateNotEmpty: true,
          emptyMessage: "Please enter a CCR limit",
          allowFloat: true,
        }
      )),
      (this.alocLimitEditor = this._createPositiveIntegerEditor(
        "ALOC Limit",
        "alocLimit",
        "ALOC limit",
        "Enter a number",
        0,
        9999999,
        {
          validateNotEmpty: true,
          emptyMessage: "Please enter an ALOC limit",
          allowFloat: true,
        }
      )),
      (this.sdLimitEditor = this._createPercentEditor(
        "SD Limit",
        "sdLimit",
        "SD limit percent value",
        "Enter a percent value",
        1,
        100,
        {
          validateNotEmpty: true,
          emptyMessage: "Please enter a SD limit",
          allowFloat: true,
        }
      )),
      (this.sdLengthEditor = this._createPositiveIntegerEditor(
        "SD Length",
        "sdLength",
        "SD length",
        "Enter a number",
        0,
        9999999,
        {
          validateNotEmpty: true,
          emptyMessage: "Please enter a SD length",
          allowFloat: true,
        }
      )),
    ];
  }

  _createNameEditor() {
    return new TextPropertyEditor({
      label: "Name",
      propertyName: "name",
      placeholder: "Group Name",
      className: "readonly-no-border",
      validateOptions: {
        notEmpty: true,
        emptyMessage: "Please enter a name",
      },
    });
  }

  _createStatusEditor() {
    return new SelectPropertyEditor({
      label: "Current Status",
      propertyName: "status",
      placeholder: "Select current status",
      className: "readonly-no-border termination-status",
      validateOptions: {
        notEmpty: true,
        emptyMessage: "Please select a current status",
      },
    });
  }

  _createLoadDistributionAlgorithmEditor() {
    const { loadDistributionAlgorithms } = TerminationVendorTrunkGroup;

    return new SelectPropertyEditor({
      label: "Route Algorithm",
      propertyName: "loadDistributionAlgorithm",
      validateOptions: {
        notEmpty: true,
        emptyMessage: PropertyEditorBuilder.getValueEmptyMessage(
          "an algorithm",
          true
        ),
      },
      selectOptions: {
        selectOptions: [
          ValueSelectOption.fromEnumItem(loadDistributionAlgorithms.roundRobin),
          ValueSelectOption.fromEnumItem(
            loadDistributionAlgorithms.randomDestination
          ),
          ValueSelectOption.fromEnumItem(loadDistributionAlgorithms.priority),
        ],
        valuesAreEqualComparer: ValueSelectOption.valuesAreEqual,
      },
    });
  }

  _createVendorUniqueIdentifierEditor() {
    return new TextPropertyEditor({
      label: "Vendor Unique Identifier",
      propertyName: "vendorUniqueIdentifier",
      placeholder: "Enter a unique identifier",
    });
  }

  _createVendorRoundingDigitsEditor() {
    return new TextPropertyEditor({
      label: "Vendor Rounding Digits",
      propertyName: "vendorRoundingDigits",
      placeholder: "Enter a vendor rounding digit",
      validateOptions: {
        notEmpty: true,
        emptyMessage: "Please enter a number for vendor rounding digits",
        custom(value) {
          const numValue = +value;
          if (typeof numValue !== "number" || isNaN(numValue)) {
            return "Vendor Rounding Digits needs to be a number";
          }

          if (numValue < 1 || numValue > 59) {
            return "Vendor Rounding Digits needs to be a number between 1 and 59";
          }
        },
      },
    });
  }

  _createVendorEditor() {
    return new SelectPropertyEditor({
      label: "Vendor",
      propertyName: "vendor",
      showSearchBox: true,
      validateOptions: {
        notEmpty: true,
        emptyMessage: "Please select a vendor",
      },
    });
  }

  _createVendorLoadBalancerEditor() {
    return new SelectPropertyEditor({
      label: "Vendor Load Balancer",
      propertyName: "vendorLoadBalancer",
      validateOptions: {
        notEmpty: false,
      },
    });
  }

  _createNotesEditor() {
    return new TextPropertyEditor({
      label: "Notes",
      propertyName: "notes",
      placeholder: "Enter any notes here...",
      multiline: true,
      rows: 5,
    });
  }

  _createCpsLimitEditor() {
    return new TextPropertyEditor({
      label: "CPS Limit",
      propertyName: "cpsLimit",
      getValueOptions: {
        tryConvertToNumber: true,
      },
      validateOptions: {
        notEmpty: true,
        emptyMessage: "Please enter a CPS Limit",
        custom: (value) => {
          let totalActiveCPSLimitsOfIPAddresses = 0;

          if (!this._isAddModal) {
            for (let ip of this._tableRow._dataObject.ipAddresses) {
              if (ip.getStatus().getId() === 1) {
                totalActiveCPSLimitsOfIPAddresses += ip.cpsLimit;
              }
            }

            if (totalActiveCPSLimitsOfIPAddresses > value) {
              return "CPS Limit needs to be equal to or greater than the total of its IP limits.";
            }
          }
        },
      },
    });
  }

  _createTrafficClassesEditor() {
    return new SelectPropertyEditor({
      label: "Traffic Classes",
      propertyName: "trafficClasses",
      multiselect: true,
      validateOptions: {
        notEmpty: true,
        emptyMessage: "Please select at least one traffic class",
      },
    });
  }

  _createTenantEditor() {
    return new SelectPropertyEditor({
      label: "Tenant",
      propertyName: "tenant",
      placeholder: "Select tenant",
    });
  }

  _createBillingIncrementsInitialEditor() {
    return new TextPropertyEditor({
      label: "Initial Billing Increments",
      propertyName: "billingIncrementsInitial",
      placeholder: "Enter initial billing increments",
      getValueOptions: {
        tryConvertToNumber: true,
      },
      validateOptions: {
        notEmpty: true,
        emptyMessage: "Please enter an initial billing increment",
      },
    });
  }

  _createBillingIncrementsContinuousEditor() {
    return new TextPropertyEditor({
      label: "Continuous Billing Increments",
      propertyName: "billingIncrementsContinuous",
      placeholder: "Enter continuous billing increments",
      getValueOptions: {
        tryConvertToNumber: true,
      },
      validateOptions: {
        notEmpty: true,
        emptyMessage: "Please enter an initial billing increment",
      },
    });
  }

  _createBooleanEditor(label, propertyName) {
    return new SelectPropertyEditor({
      label: label,
      propertyName: propertyName,
      selectOptions: {
        selectOptions: [
          new ValueSelectOption("True", true),
          new ValueSelectOption("False", false),
        ],
        valuesAreEqualComparer: ValueSelectOption.valuesAreEqual,
      },
    });
  }

  _createPositiveIntegerEditor(
    label,
    propertyName,
    description,
    placeholder,
    minValue,
    maxValue,
    validators
  ) {
    return new TextPropertyEditor(
      _.extend(
        {
          label: label,
          propertyName: propertyName,
          placeholder: placeholder,
        },
        PropertyEditorBuilder.getNumberEditorOptions(
          description,
          minValue,
          maxValue,
          validators
        )
      )
    );
  }

  _createPercentEditor(
    label,
    propertyName,
    description,
    placeholder,
    minValue,
    maxValue,
    validators
  ) {
    return new TextPropertyEditor(
      _.extend(
        {
          label: label,
          propertyName: propertyName,
          placeholder: placeholder,
          className: "readonly-no-border",
        },

        this.getNumberEditorOptions(description, minValue, maxValue, validators)
      )
    );
  }

  getNumberEditorOptions(description, ...args) {
    let minValue, maxValue, options;

    if (args.length) {
      if (_.isObject(args[0])) options = args[0];
      else [minValue, maxValue, options] = args;
    }

    /*
    options can contain validateNotEmpty, allowFloat, validate, validateEmpty, minValueText, maxValueText
    */
    options = options || {};

    const minValueText = options.minValueText || minValue;
    const maxValueText = options.maxValueText || maxValue;

    return {
      getValueOptions: {
        trimValue: false,
        convertEmptyToNull: true,
        tryConvertToNumber: options.allowFloat ? { allowFloat: true } : true,
      },
      validateOptions: {
        notEmpty: options.validateNotEmpty,
        emptyMessage: PropertyEditorBuilder.getValueEmptyMessage(
          "the " + description
        ),
        custom: function (value, data) {
          if (Helpers.trimAsString(value).length === 0) {
            if (options.validateEmpty) {
              let validationResult = options.validateEmpty(data);
              if (validationResult) return validationResult;
            }
          } else {
            if (typeof value !== "number")
              return (
                "The " +
                description +
                ' should only contain a percent number without the percent sign "%"'
              );

            if (Helpers.isDefinedAndNotNull(minValue) && value < minValue)
              return (
                "Please enter the percent value.  The minimum allowed " +
                description +
                " is " +
                minValueText
              );

            if (Helpers.isDefinedAndNotNull(maxValue) && value > maxValue)
              return (
                "The maximum allowed " + description + " is " + maxValueText
              );

            if (options.largerThanZero && value <= 0)
              return "The " + description + " should be larger than zero";

            if (options.validate) {
              let validationResult = options.validate(value, data);
              if (validationResult) return validationResult;
            }
          }
        },
      },
    };
  }

  addVendor(vendor) {
    this.vendors = _.sortBy([vendor].concat(this.vendors), function (i) {
      return i.name.toLowerCase();
    });

    this.vendorEditor.setSelectOptions(
      DataObjectSelectOption.fromDataObjects(this.vendors),
      DataObjectSelectOption.valuesAreEqual
    );
  }

  removeVendor(vendor) {
    let newVendorList = [];
    for (let v of this.vendors) {
      if (v.getId() !== vendor.getId()) {
        newVendorList.push(v);
      }
    }
    this.vendors = newVendorList;
    this.vendorEditor.setSelectOptions(
      DataObjectSelectOption.fromDataObjects(this.vendors),
      DataObjectSelectOption.valuesAreEqual
    );
  }

  updateVendor(vendor) {
    for (let index in this.vendors) {
      if (this.vendors[index].getId() === vendor.getId()) {
        this.vendors[index] === vendor;
      }
    }

    this.vendorEditor.setSelectOptions(
      DataObjectSelectOption.fromDataObjects(this.vendors),
      DataObjectSelectOption.valuesAreEqual
    );
  }

  _beforeShow(isAddModal) {
    this._isAddModal = isAddModal;
  }

  update(isAdmin, statuses, trafficClasses, vendors, vlbs) {
    this._adminMode = isAdmin;
    this.statuses = statuses;

    this.vendors = _.sortBy(vendors, function (i) {
      return i.name.toLowerCase();
    });

    this.trafficClasses = _.sortBy(trafficClasses, function (trafficClass) {
      return trafficClass.name.toLowerCase();
    });

    this.setAddObjectTemplate(
      TerminationTrunkGroupStatus.getDefaultStatus(statuses)
    );

    const renderedStatuses = _.map(
      statuses,
      (status) =>
        new DataObjectSelectOption(status, {
          optionClass: TerminationTableHelpers.getStatusClassName(
            status.getId()
          ),
        })
    );

    this.statusEditor.setSelectOptions(
      renderedStatuses,
      DataObjectSelectOption.valuesAreEqual
    );

    this.trafficClassesEditor.setSelectOptions(
      DataObjectSelectOption.fromDataObjects(this.trafficClasses),
      DataObjectSelectOption.valuesAreEqual
    );

    this.vendorEditor.setSelectOptions(
      DataObjectSelectOption.fromDataObjects(this.vendors),
      DataObjectSelectOption.valuesAreEqual
    );
    
    this.vendorLoadBalancerEditor.setSelectOptions(
      [
        new ValueSelectOption("None", null, {
          spanClass: "light-text",
          title: "None",
          appendDivider: true,
        }),
      ].concat(DataObjectSelectOption.fromDataObjects(vlbs)),
      DataObjectSelectOption.valuesAreEqual
    );
  }
}

export default TerminationVendorTrunkGroupEditorModal;

```
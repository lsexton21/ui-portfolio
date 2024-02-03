## Custom Form Layout, Complex Validations, and Custom Drag and Drop Feature

This is an example of a form to create traffic rules that had some complexity. First, I gave it a rather peculiar format for better UX experience.  Second, the validations were highly regulated (every rule that was created dynamically changed the range of validations for the next rule).  And third, nothing too fancy, but I added a little drag and drop feature for reordering the rules (which, of course, would change all the validations of each).

**See below for code**

![Add Robocop Rules](/assets/robocopRulesAdd.png)
![Add Robocop Rules](/assets/robocopRulesReorder.png)


---


#### This is the code that contains most of the logic for this editor modal.  It's long simply because it combines both the table and editor in one file, and it contains a lot of unique code to the app. If I were to repeat, I would refactor.

```
import Modal from "../Modal";
import ModalCoverMixin from "../ModalCoverMixin";
import TerminationRobocopRule from "../../data/termination/TerminationRobocopRule";
import ObjectEditor from "../../components/ObjectEditor";
import SelectPropertyEditor from "../../components/SelectPropertyEditor";
import TextPropertyEditor from "../../components/TextPropertyEditor";
import PropertyEditorBuilder from "../../builders/PropertyEditorBuilder";
import ValueSelectOption from "../../components/ValueSelectOption";
import TerminationRoutingTemplate from "../../data/termination/TerminationRoutingTemplate";
import App from "../../App";
import ElementHelpers from "../../helpers/ElementHelpers";
import Helpers from "../../helpers/Helpers";
import MessageModal from "../MessageModal";
import TerminationCurrentRobocopRulesSet from "../../data/termination/TerminationCurrentRobocopRulesSet";
import ConfirmationModal from "../ConfirmationModal";
import HtmlTemplates from "../../templates/HtmlTemplates";

class TerminationRobocopModal extends ModalCoverMixin(Modal) {
  constructor() {
    super({
      dataTitle: "Robocop Rules",
      dialogClass: "termination-robocop-modal",
      body:
        '<div class="robocop-rules-list-container">' +
        '<div class="robocop-rules-btns">' +
        '<button class="add-robocop-rule-btn btn" type="button"><i class="fa fa-plus"></i>Add Rule</button>' +
        '<button class="reorder-rules-btn btn" type="button"><i class="fa fa-random"></i>Reorder Rules</button>' +
        "</div>" +
        '<div class="rules-list-container">' +
        '<ol class="current-robocop-rules-list">' +
        "</ol>" +
        "</div>" +
        "</div>",
      footer: '<div class="source-update-text"><em>Sources are determined over a 72 hour period and are updated hourly</em></div><button type="button" class="btn btn-primary accept-btn">Save</button>'
      + ' <button type="button" class="btn btn-default" data-dismiss="modal">Cancel</button>'
    });

    this.on("cancel", () => {
      this._$modal.find(".current-robocop-rules-list").empty();
      this._currentRobocopRulesSet = null;
    });

    this._initHeaderBtns();

    const $footer = this._$modal.find(".modal-footer");
    $footer.addClass("robocop-footer");
  }

  _initHeaderBtns() {
    this._$addRuleBtn = this._$modal.find(".add-robocop-rule-btn");
    this._$reorderRulesBtn = this._$modal.find(".reorder-rules-btn");

    this._$addRuleBtn.on("click", () => {
      if (ElementHelpers.isDisabled(this._$addRuleBtn)) {
        return false;
      }
      this._initAddRuleEditors();
      this.updateAllEditors();
      const $addRobocopRuleContainer = this._$modal.find(
        ".add-robocop-rule-container"
      );

      $addRobocopRuleContainer.fadeIn(100);
      $addRobocopRuleContainer[0].scrollIntoView();

      ElementHelpers.enable(this._$addRuleBtn, false);
      ElementHelpers.enable(this._$reorderRulesBtn, false);

      this._enableAcceptButton(false);
    });

    this._$reorderRulesBtn.on("click", () => {
      if (ElementHelpers.isDisabled(this._$reorderRulesBtn)) {
        return false;
      }

      if (this.reorderingInProgress) {
        this._enableAcceptButton(true);
        this.reorderingInProgress = false;
        this._$reorderRulesBtn.html(
          '<i class="fa fa-random"></i>     Reorder Rules'
        );
        const $currentRobocopRulesList = this._$modal.find(
          ".current-robocop-rules-list"
        );

        $currentRobocopRulesList.off("dragover");
        const $currentRules = $currentRobocopRulesList
          .find("li")
          .filter(".rule-item");

        const idArray = [];
        $currentRules.each((index, element) => {
          $(element).removeClass("reordering");
          $(element).off("dragstart");
          $(element).off("dragend");
          idArray.push(parseInt($(element).attr("ruleId")));
        });

        this._currentRobocopRulesSet.sortUpdatedRobocopRulesByIds(idArray);

        ElementHelpers.enable(this._$addRuleBtn, true);
        this._enableAcceptButton(true);
        return false;
      } else {
        this.reorderingInProgress = true;
        this._$reorderRulesBtn.html(
          '<div class="confirm-reordering"><i class="fa fa-check"></i>    Confirm Reorder</div>'
        );
        const $currentRobocopRulesList = this._$modal.find(
          ".current-robocop-rules-list"
        );

        const initSortableList = (event) => {
          event.preventDefault();
          const $draggingItem = $currentRobocopRulesList.find(".dragging");

          const $siblings = $currentRobocopRulesList
            .find("li:not(.dragging)")
            .filter(".rule-item")
            .toArray();

          let $nextSibling = $siblings.find((sibling) => {
            const $sibling = $(sibling);
            const offsetTop = Math.round($sibling.offset().top);
            const offsetHeight = Math.round($sibling.outerHeight(true));
            return event.clientY <= offsetTop + offsetHeight / 2;
          });

          if (!$nextSibling) {
            $draggingItem.insertAfter($siblings[$siblings.length - 1]);
          } else {
            $draggingItem.insertBefore($nextSibling);
          }
        };

        $currentRobocopRulesList.on("dragover", initSortableList);
        const $currentRules = $currentRobocopRulesList
          .find("li")
          .filter(".rule-item");

        $currentRules.each((index, element) => {
          $(element).addClass("reordering");
          $(element).on("dragstart", () => {
            setTimeout(() => $(element).addClass("dragging"), 0);
          });
          $(element).on("dragend", () => {
            $(element).removeClass("dragging");
          });
        });

        ElementHelpers.enable(this._$addRuleBtn, false);
        this._enableAcceptButton(false);
      }
    });
  }

  _enableReorderingButton() {
    const $currentRobocopRulesList = this._$modal.find(
      ".current-robocop-rules-list"
    );
    const $allRules = $currentRobocopRulesList
      .find("li")
      .filter(".rule-item")
      .toArray();

    ElementHelpers.enable(this._$reorderRulesBtn, $allRules.length > 1);
  }

  _initAddRuleEditors() {
    const $addRobocopRuleContainer = $(
      '<li class="add-robocop-rule-container">' +
      '<div class="header-and-btns-container">' +
      '<h4 class="modal-title">Adding a New Rule</h4>' +
      '<div class="inline-row-editor-buttons">' +
      '<div class="btn-group">' +
      '<button class="btn btn-default btn-xs confirm-robocop-rule-btn">' +
      '<i class="fa fa-check" aria-hidden="true"></i>' +
      "</button>" +
      '<button class="btn btn-default btn-xs cancel-robocop-rule-btn">' +
      '<i class="fa fa-undo" aria-hidden="true"></i>' +
      "</button>" +
      "</div>" +
      "</div>" +
      '</div>' +
        '<div class="add-rule-details-container">' +
        '<div class="source-and-destination-container">' +
        '<ul class="source-details rule-details">Source Parameters: ' +
        '<li class="detail-item">Using this identifier: <select id="identifier"></select></li>' +
        '<li class="detail-item">If the <select id="metric"></select></li>' +
        '<li class="detail-item">Is between <input id="minValue" type="text" placeholder="min value"/> and <input id="maxValue" type="text" placeholder="max value"/><span class="value-units"> seconds</span></li>' +
        '<li class="detail-item">With a minimum of <input id="min-call-attempts" type="text" placeholder="min value"/> call attempts</li>' +
        "</ul>" +
        '<ul class="destination-add-details rule-details"><input class="form-check-input" type="checkbox" id="destinationCheckbox">Add Destination Parameter (optional):' +
        '<li class="detail-item destination-type-label">Destination <select id="destination-type"></select></li>' +
        '<li class="detail-item spid-label">SPID: <input id="spid" type="text" placeholder="Enter SPID"/></li>' +
        "</ul>" +
        "</div>" +
        '<div class="action-and-notes-container">' +
        '<ul class="action-details rule-details">Result:' +
        '<li class="detail-item action-item">The call will be <select id="action"></select></li>' +
        "</ul>" +
        '<ul class="rule-details monitor-details"><input class="form-check-input" type="checkbox" id="monitorCheckbox">Monitor Only</ul>' +
        '<div class="notes-container"><span>Notes:</span>' +
        '<textarea rows="5" id="notes" placeholder="Enter any notes here..."></textarea>' +
        "</div>" +
        "</div>" +
        "</div>" +
        "</li>"
    );

    const $currentRobocopRulesList = this._$modal.find(
      ".current-robocop-rules-list"
    );

    if (this._currentRobocopRulesSet.getCurrentRules().length === 0) {
      const $noRuleMsg = this._$modal.find(".no-rule-msg");
      $noRuleMsg.remove();
    }

    $currentRobocopRulesList.append($addRobocopRuleContainer);

    const metricEditor = this._createMetricEditor();
    const identifierEditor = this._createIdentifierEditor();
    const minValueEditor = this._createMinValueEditor();
    const maxValueEditor = this._createMaxValueEditor();
    const minCallAttemptsEditor = this._createMinCallAttemptsEditor();
    const actionEditor = this._createActionEditor();
    const notesEditor = this._createNotesEditor();

    this.metricEditor = metricEditor;
    this.identifierEditor = identifierEditor;
    this.actionEditor = actionEditor;
    this.minValueEditor = minValueEditor;
    this.maxValueEditor = maxValueEditor;

    const destinationTypeEditor = this._createDestinationTypeEditor();
    const spidEditor = this._createSpidEditor();

    this.destinationTypeEditor = destinationTypeEditor;
    this.spidEditor = spidEditor;

    this.objectEditor = new ObjectEditor([
      metricEditor,
      identifierEditor,
      minValueEditor,
      maxValueEditor,
      minCallAttemptsEditor,
      actionEditor,
      destinationTypeEditor,
      spidEditor,
      notesEditor,
    ]);

    this.metricEditor.on("changed", this._setValueUnits.bind(this));

    const $destinationCheckbox = this._$modal.find("#destinationCheckbox");
    $destinationCheckbox.on("change", () => {
      this._destinationEditorsEnabler();
    });

    this.destinationTypeEditor.getInputElement().parent().addClass("dropup");

    this.destinationTypeEditor.on("changed", () => {
      this._destinationEditorsEnabler();
    });
    this._destinationEditorsEnabler();

    const $confirmRuleBtn = this._$modal.find(".confirm-robocop-rule-btn");
    $confirmRuleBtn.on("click", () => {
      const editorValues = this.objectEditor.getValues();

      if (!editorValues) {
        return false;
      }

      const newRule = new TerminationRobocopRule({
        priority: this._currentRobocopRulesSet.getCurrentRules().length + 1,
        id: Math.round(Math.random() * 1000000),
        identifier: editorValues.identifier.rawValue,
        metric: editorValues.metric.rawValue,
        min_value: editorValues.minValue,
        max_value: editorValues.maxValue,
        min_call_attempts: editorValues.minCallAttempts,
        action: editorValues.action,
        notes: editorValues.notes,
      });

      const $destinationCheckbox = this._$modal.find("#destinationCheckbox");
      const checkboxValue = $destinationCheckbox.is(":checked");

      if (checkboxValue) {
        switch (editorValues.destinationType.rawValue) {
          case "spid":
            newRule.destinationType = "spid";
            newRule.destinationValue = editorValues.destinationValue;
            break;
          case true:
            newRule.destinationType = "dnc";
            newRule.destinationValue = true;
            break;
          case false:
            newRule.destinationType = "dnc";
            newRule.destinationValue = false;
            break;
          default:
            newRule.destinationType = null;
            newRule.destinationValue = null;
        }
      } else {
        newRule.destinationType = null;
        newRule.destinationValue = null;
      }

      const $monitorCheckbox = this._$modal.find("#monitorCheckbox");
    const monitorCheckboxValue = $monitorCheckbox.is(":checked");

    if (monitorCheckboxValue) {
      newRule.monitor = true;
    } else {
      newRule.monitor = false;
    }

      const isValid =
        this._currentRobocopRulesSet.validateUniqueRobocopRule(newRule);

      if (!isValid) {
        MessageModal.show({
          title: "Error",
          text: "There is already a rule with the same identifier, metric, overlapping value range, and destination parameters.  Please change one of these values and try again.",
          isError: true,
        });
      } else {
        $addRobocopRuleContainer.fadeOut(100);
        $addRobocopRuleContainer.remove();

        $currentRobocopRulesList.append(
          newRule.getRobocopRuleHtml(this.routingTemplates)
        );

        this._currentRobocopRulesSet.addRobocopRule(newRule);

        this._initDeleteRuleButtons();
        this._initNotesBtns();

        this._enableReorderingButton();
        ElementHelpers.enable(this._$addRuleBtn, true);
        this._enableAcceptButton(true);
      }
    });

    const $cancelRuleBtn = this._$modal.find(".cancel-robocop-rule-btn");
    $cancelRuleBtn.on("click", () => {
      $addRobocopRuleContainer.fadeOut(200);

      setTimeout(() => {
        this.objectEditor.clear();
        this._enableAcceptButton(true);
        this._enableReorderingButton();
        ElementHelpers.enable(this._$addRuleBtn, true);
        $addRobocopRuleContainer.remove();
      }, 200);

      if (this._currentRobocopRulesSet.getCurrentRules().length === 0) {
        setTimeout(() => {
          const $noRuleMsg = $(
            '<div class="no-rule-msg">There are no current rules set for this traffic class.</div>'
          );
          $currentRobocopRulesList.append($noRuleMsg);
        }, 200);
      }
    });
  }

  _destinationEditorsEnabler() {
    const $destinationTypeLabel = this._$modal.find(".destination-type-label");

    const $spidLabel = this._$modal.find(".spid-label");

    const $destinationCheckbox = this._$modal.find("#destinationCheckbox");
    const checkboxValue = $destinationCheckbox.is(":checked");

    if (checkboxValue) {
      this.destinationTypeEditor.setReadOnly(false);
      $destinationTypeLabel.animate({ opacity: 1 }, 400);
      this.destinationTypeEditor.setValidateNotEmpty(true);

      if (this.destinationTypeEditor.getValue().rawValue === "spid") {
        this.spidEditor.setReadOnly(false);
        $spidLabel.animate({ opacity: 1 }, 400);
        this.spidEditor.setValidateNotEmpty(true);
      } else {
        this.spidEditor.setReadOnly(true);
        $spidLabel.animate({ opacity: 0.3 }, 400);
        this.spidEditor.setValidateNotEmpty(false);
      }
    } else {
      this.destinationTypeEditor.setReadOnly(false);
      $destinationTypeLabel.animate({ opacity: 0.3 }, 400);
      this.destinationTypeEditor.setValidateNotEmpty(false);
      this.spidEditor.setReadOnly(true);
      $spidLabel.animate({ opacity: 0.3 }, 400);
      this.spidEditor.setValidateNotEmpty(false);
    }
  }

  _setValueUnits() {
    const $valueUnits = this._$modal.find(".value-units");
    if (
      this.metricEditor.getValue() &&
      this.metricEditor.getValue().rawValue === "duration"
    ) {
      $valueUnits.text(" seconds");
    } else {
      $valueUnits.text(" percent");
    }
  }

  _createMetricEditor() {
    const $select = this._$modal.find("#metric");

    return new SelectPropertyEditor($select, $select, {
      propertyName: "metric",
      placeholder: "select a metric",
      validateOptions: {
        notEmpty: true,
        emptyMessage: PropertyEditorBuilder.getValueEmptyMessage(
          "metric",
          true
        ),
      },
    });
  }

  _createIdentifierEditor() {
    const $select = this._$modal.find("#identifier");
    return new SelectPropertyEditor($select, $select, {
      propertyName: "identifier",
      validateOptions: {
        notEmpty: true,
        emptyMessage: PropertyEditorBuilder.getValueEmptyMessage(
          "identifier",
          true
        ),
      },
    });
  }

  _createMinValueEditor() {
    const $input = this._$modal.find("#minValue");
    return new TextPropertyEditor($input, $input, {
      propertyName: "minValue",
      validateOptions: {
        notEmpty: true,
        emptyMessage: "Please enter a minimum value",
        custom: (value) => {
          if (isNaN(+value)) {
            return "Please enter a valid number";
          }

          if (
            this.maxValueEditor.getValue() &&
            +value >= this.maxValueEditor.getValue()
          ) {
            return "Minimum value must be less than maximum value";
          }
        },
      },
    });
  }

  _createMaxValueEditor() {
    const $input = this._$modal.find("#maxValue");
    return new TextPropertyEditor($input, $input, {
      propertyName: "maxValue",
      validateOptions: {
        notEmpty: true,
        emptyMessage: "Please enter a maximum value",
        custom: (value) => {
          if (isNaN(+value)) {
            return "Please enter a valid number";
          }
        },
      },
    });
  }

  _createMinCallAttemptsEditor() {
    const $input = this._$modal.find("#min-call-attempts");
    return new TextPropertyEditor($input, $input, {
      propertyName: "minCallAttempts",
      validateOptions: {
        notEmpty: true,
        emptyMessage: PropertyEditorBuilder.getValueEmptyMessage(
          "minimum call attempt",
          true
        ),
        custom: (value) => {
          if (isNaN(+value)) {
            return "Please enter a valid number";
          }
        },
      },
    });
  }

  _createDestinationTypeEditor() {
    const $select = this._$modal.find("#destination-type");
    return new SelectPropertyEditor($select, $select, {
      propertyName: "destinationType",
      validateOptions: {
        notEmpty: true,
        emptyMessage: PropertyEditorBuilder.getValueEmptyMessage(
          "destination type",
          true
        ),
      },
      // selectOptions: {
      //   selectOptions: [
      //     ValueSelectOption.fromEnumItem(destinationTypes.spid),
      //     ValueSelectOption.fromEnumItem(destinationTypes.onDnc),
      //     ValueSelectOption.fromEnumItem(destinationTypes.offDnc),
      //   ],
      //   valuesAreEqualComparer: ValueSelectOption.valuesAreEqual,
      // },
    });
  }

  _createSpidEditor() {
    const $input = this._$modal.find("#spid");
    return new TextPropertyEditor($input, $input, {
      propertyName: "destinationValue",
      validateOptions: {
        notEmpty: true,
        emptyMessage: "Please enter a spid value",
      },
    });
  }

  _createActionEditor() {
    const $select = this._$modal.find("#action");
    return new SelectPropertyEditor($select, $select, {
      propertyName: "action",
      validateOptions: {
        notEmpty: true,
        emptyMessage: PropertyEditorBuilder.getValueEmptyMessage(
          "action",
          true
        ),
      },
    });
  }

  _createNotesEditor() {
    const $textarea = this._$modal.find("#notes");
    return new TextPropertyEditor($textarea, $textarea, {
      propertyName: "notes",
      placeholder: "Enter any notes here...",
    });
  }

  async fetchCurrentRobocopRules(trafficClassId) {
    this._currentRobocopRulesSet =
      await TerminationCurrentRobocopRulesSet.getAll(trafficClassId);

    // this._currentRobocopRulesSet =
    //   TerminationCurrentRobocopRulesSet.getRobocopDummyData(trafficClassId);

    const $currentRobocopRulesList = this._$modal.find(
      ".current-robocop-rules-list"
    );
    const $noRuleMsg = $(
      '<div class="no-rule-msg">There are no current rules set for this traffic class.</div>'
    );

    if (Helpers.isEmpty(this._currentRobocopRulesSet.getCurrentRules())) {
      $currentRobocopRulesList.append($noRuleMsg);
    } else {
      this._currentRobocopRulesSet.getCurrentRules().forEach((rule) => {
        $currentRobocopRulesList.append(
          rule.getRobocopRuleHtml(this.routingTemplates)
        );
      });

      this._initDeleteRuleButtons();
      this._initNotesBtns();
    }
  }

  async updateAllEditors() {
    const destinationTypes = TerminationRobocopRule.destinationTypes;
    this.destinationTypeEditor.setSelectOptions(
      [
        ValueSelectOption.fromEnumItem(destinationTypes.spid),
        ValueSelectOption.fromEnumItem(destinationTypes.onDnc),
        ValueSelectOption.fromEnumItem(destinationTypes.offDnc),
      ],
      ValueSelectOption.valuesAreEqual
    );

    const metrics = TerminationRobocopRule.metrics;
    this.metricEditor.setSelectOptions(
      [
        ValueSelectOption.fromEnumItem(metrics.averageCallDuration),
        ValueSelectOption.fromEnumItem(metrics.notFound),
        ValueSelectOption.fromEnumItem(metrics.requestTerminated),
        ValueSelectOption.fromEnumItem(metrics.generalBlock),
        ValueSelectOption.fromEnumItem(metrics.endUserBlock),
        ValueSelectOption.fromEnumItem(metrics.intermediaryBlock),
      ],
      ValueSelectOption.valuesAreEqual
    );

    const identifiers = TerminationRobocopRule.identifiers;
    this.identifierEditor.setSelectOptions(
      [
        ValueSelectOption.fromEnumItem(identifiers.ani),
        ValueSelectOption.fromEnumItem(identifiers.dnis),
        ValueSelectOption.fromEnumItem(identifiers.customerMediaIp),
        ValueSelectOption.fromEnumItem(identifiers.certificate),
        ValueSelectOption.fromEnumItem(identifiers.customerUserAgent),
      ],
      ValueSelectOption.valuesAreEqual
    );

    const selectOptions = [new ValueSelectOption("Blocked", 0)].concat(
      this.routingTemplates.map(
        (template) =>
          new ValueSelectOption(
            `Rerouted to ${template.getName()}`,
            template.getId()
          )
      )
    );

    this.actionEditor.setSelectOptions(
      selectOptions,
      ValueSelectOption.valuesAreEqual
    );
  }

  async submitChanges(modal) {
    try {
      modal._showCover(true);
      const response = await modal._currentRobocopRulesSet.commitRobocopRules();
      if (response) {
        MessageModal.show({
          title: "Rules Updated!",
          text: "Your Robocop Rules have been updated successfully.",
          isError: true,
        });
      }
      this._enableAcceptButton(false);
    } catch (error) {
      MessageModal.show({
        title: "Error",
        text: "There was an error processing your request.  Please try again.  If the problem persists, please contact the DEV team.",
        isError: true,
        devErrorMsg: error,
      });
    } finally {
      modal._hideCover(true);
    }
  }

  _initDeleteRuleButtons() {
    const $deleteRuleBtns = this._$modal.find(".delete-rule-btn");

    $deleteRuleBtns.each((index, element) => {
      $(element).on("click", () => {
        const confirmDeleteModal = new ConfirmationModal();
        confirmDeleteModal.showModal({
          title: "Delete Rule",
          text: "Are you sure you want to delete this rule?",
          subText:
            "Remember, your changes will not be saved until you also click 'Save'.",
          acceptBtnTitle: "Delete",
          acceptBtnClass: "btn-danger",
          acceptBtnClickedCallback: () => {
            this._enableAcceptButton(true);
            const ruleId = parseInt($(element).attr("ruleId"));
            this._currentRobocopRulesSet.removeRobocopRuleById(ruleId);

            const $currentRobocopRulesList = this._$modal.find(
              ".current-robocop-rules-list"
            );

            const $ruleToDelete = $currentRobocopRulesList.find(
              `li[ruleId=${ruleId}]`
            );
            $ruleToDelete.fadeOut(200);
            setTimeout(() => $ruleToDelete.remove(), 200);

            if (this._currentRobocopRulesSet.getCurrentRules().length === 0) {
              const $noRuleMsg = $(
                '<div class="no-rule-msg">There are no current rules set for this traffic class.</div>'
              );
              setTimeout(
                () => $currentRobocopRulesList.append($noRuleMsg),
                200
              );
            }
          },
        });
      });
    });
  }

  _initNotesBtns() {
    const $NotesBtns = this._$modal.find(".notes-btn");

    $NotesBtns.each((index, element) => {
      const ruleId = parseInt($(element).attr("ruleId"));
      const rule = this._currentRobocopRulesSet
        .getCurrentRules()
        .find((rule) => rule.getId() === ruleId);

      if (rule.getNotes()) {
        $(element).tooltip({
          placement: "top",
          title: "View Notes",
          template: HtmlTemplates.getTooltipHtml("blue-tooltip"),
        });

        $(element).on("click", () => {
          MessageModal.show({
            title: "Notes",
            text: rule.getNotes(),
            isError: false,
          });
        });
      } else {
        $(element).addClass("disabled");
        $(element).fadeTo(0, 0);
      }
    });
  }

  async show(trafficClass) {
    this.trafficClass = trafficClass;
    this._show({
      title: "Robocop Rules",
      acceptBtnTitle: "Save",
      acceptBtnClickedCallback: () => {
        this.submitChanges(this);
        return false;
      },
    });
    this._showCover(true);
    this.routingTemplates = await TerminationRoutingTemplate.getAll(
      App.getAuth()
    );
    if (Helpers.isUndefinedOrNull(this._currentRobocopRulesSet)) {
      await this.fetchCurrentRobocopRules(trafficClass.getId());

      this._enableAcceptButton(false);
      ElementHelpers.enable(this._$addRuleBtn, true);
      this._enableReorderingButton();
    }
    this._hideCover(true);
  }
}

export default TerminationRobocopModal;

```
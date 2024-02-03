## Converting Formdata To HTML

This is an example of when I needed to creat a tool that would allow our admin to easy view routing data from the API in HTML.  

**See below for code**

![Routing lookup page](/assets/routingLookup.png)


---


#### This code snippet is a function I wrote (from scratch) which converted the raw data from our API into HTML, so that I could add a little styling for easier viewing

```
createHtmlListItemsFromDataObject: function (data) {
    const $resultList = $("<ul class='result-list'></ul>");

    function addListItem($resultList, data, childNum = 0) {
      const parentData = data;
      const parentKeys = Object.keys(parentData);

      for (let i = 0; i < parentKeys.length; i++) {
        const childData = parentData[parentKeys[i]];

        if (Helpers.isObject(childData)) {
          const $resultParentItem = $(
            `<li class="child-item-${childNum}">${parentKeys[i]}<ul class="child-list-${childNum}"></ul></li>`
          );
          $resultList.append($resultParentItem);
          const $childList = $resultParentItem.find(`.child-list-${childNum}`);
          const newChildNum = childNum + 1;
          addListItem($childList, childData, newChildNum);
        } else {
          if (childData === null) {
            const $resultParentItem = $(
              `<li class="child-item-${childNum}">${
                parentKeys[i]
              }: <span class="child-item-${childNum + 1}">Null</span></li>`
            );
            $resultList.append($resultParentItem);
          } else if (childData === undefined) {
            const $resultParentItem = $(
              `<li class="child-item-${childNum}">${
                parentKeys[i]
              }: <span class="child-item-${childNum + 1}">Undefined</span></li>`
            );
            $resultList.append($resultParentItem);
          } else {
            const $resultParentItem = $(
              `<li class="child-item-${childNum}">${
                parentKeys[i]
              }: <span class="child-item-${
                childNum + 1
              }">${childData}</span></li>`
            );
            $resultList.append($resultParentItem);
          }
        }
      }
    }

    addListItem($resultList, data);
    
    return $resultList;
  },
```
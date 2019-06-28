# Entities-in-Draft-js

In Draft-js I have been exploring ways to annotate pieces of data so that I can use `convertToRaw` and send this to my server. One way I recently explored was using a Draft Entity to store meta-data on the first content block which would be the title. For this specific case it may be best to use the `Data` property of the ContentBlock, but I wanted to go over an example of using the `EditorState.createWithContent()` method. One case of using `createWithContent` is if we would like our first block to have the type of a heading when the user starts typing. One thing to note about this particular way of handling the intended functionality is that the block type will carry down to other blocks.

## The custom Block
We will start with the block that we plan to use in `convertFromRaw`.
```
const myBlock = {
  "entityMap": {
    "0": {
        "type": "TITLE",
        "mutability": "MUTABLE",
        "data": {
         "title": ""
        }
       }
  },
  "blocks": [
      {
          "key": "5h45l",
          "text": "",
          "type": "header-one",
          "depth": 0,
          "entityRanges": [
            {
             "offset": 0,
             "length": 0,
             "key": "0"
            }
           ],
          "inlineStyleRanges": [],
          "data": {}
      }
  ]
}
```
Here is an Object called myBlock that has 2 keys. One is entityMap with the value being an object of entities, and the other is blocks whith a value of an array of blocks. EntityMap has one entity object that contains the properties type, mutability, and data. Blocks has one block object with the specified type we want (header-one) along with the entityRanges array and the other block properties. EntityRanges array tells us the properties of the specified entity along with the key of the entity to be applied.

## Convert and State
The next steps are to convertFromRaw the declare our State.
```
const conState = convertFromRaw(myBlock)

this.state = {editorState: EditorState.createWithContent(conState)};
```
with the EditorState created the entity we had in myBlock will also be created.

## Replace and ApplyEntity
Even though our Entity has been created and will show up if we if we use `getLastCreatedEntityKey` it will not be added to the Entity map. This will be problematic when we want to send the raw content to the server. To fix this we will use the Draft-js Modifier Module which has the method `applyEntity`. Using this method is necessary because the immutable EditorState is packed with more immutable objects and are deeply nested. One being the CharacterMetaData which is an object containing inline style and entity data for each individual character. So the `applyEntity()` method will go deep and help ensure the nested objects know about the entity. Before we apply the new entity we need to use the `replaceEntityData()` method called on the ContentState to keep the Entity up to date. This method takes the entity key as the first argument along with an object containing the intended changes as the second. Together it will look something like this:
```
const contentState = editorState.getCurrentContent()
const firstBlock = contentState.getFirstBlock()
const entityKey = contentState.getLastCreatedEntityKey()
const selection = editorState.getSelection()
const newSelection = selection.merge({anchorOffset: 0, focusOffset: firstBlock.getLength()})

const conStateEntityData = contentState.replaceEntityData("1", {title: firstBlock.getText()})

const contentWithTitle = Modifier.applyEntity(
    conStateEntityData,
    newSelection,
    entityKey
)

const fixSelection = selection.merge({anchorOffset: firstBlock.getLength(), focusOffset: firstBlock.getLength()})

const newEditorState = EditorState.push(editorState, contentWithTitle, "apply-entity")
const changeSelectionEditorState = EditorState.forceSelection(newEditorState, fixSelection)
this.setState({editorState: changeSelectionEditorState})
```
This is a lot to process so let's make some sense of it.
The first thing to notice is how we use `merge()` on selection to alter the selectionState this is because `applyEntity()` is intended to change the data of the characters within the selectionState, so we have to pass an altered selectionState to `applyEntity` in order to affect the intended block.

Next we replace the Entity data with the text from the first block of our content(this returns a ContentState). Then we use applyEntity with the arguments of the ContentState named `conStateEntityData`, the newSelection SelectionState and the entityKey. The return of `applyEntity` is a ContentState object which we name contentWithTitle. We then have to return the SelectionState to the way that is was otherwise our block will be highlighted. All those changes are pushed to the EditorState, the we use `forceSelection` to set the SelectionState again. Lastly we set our react state.

## Things to think about
This code is the first I have got working so far for this functionality but it is most definitely not the best way to handle it. For revisions I am looking at using ContentStates `addEntity` instead of `applyEntity` to save having to reset the selectionState so many times. Something else to think about is when you want this to happen. You need a way of determining when we are making changes to the title block, and if we use `componentDidUpdate` with the intention of calling `this.onChange` inside we will get a react error. So I will be looking to get this right and simplify all of this crazyness.

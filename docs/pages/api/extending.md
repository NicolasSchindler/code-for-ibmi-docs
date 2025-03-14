It is possible to write VS Code extensions that are based on Code for IBM i. That means your extension can use the connection that the user creates in your extension. This is not an extension tutorial, but an intro on how to access the APIs available within Code for IBM i.

For example, you might be a vendor that produces lists or HTML that you'd like to be accessible from within Visual Studio Code.

# Exports

As well as the basic VS Code command API, you can get access to the Code for IBM i API with the VS Code `getExtension` API.

```
const { instance } = vscode.extensions.getExtension(`halcyontechltd.code-for-ibmi`).exports;
```

## Typings

We provide TS type definitions to make using the Code for IBM i API easier. They can be installed via `npm`:

```sh
npm i @halcyontech/vscode-ibmi-types
```

It can then be imported and used in combination with `getExtension`:

```ts
import type { CodeForIBMi } from '@halcyontech/vscode-ibmi-types';

//...

const ext = vscode.extensions.getExtension<CodeForIBMi>('halcyontechltd.code-for-ibmi');
```

**As Code for IBM i updates, the API may change.** It is recommended you always keep the types packaged updated as the extension updates, incase the API interfaces change. We plan to make the VS Code command API interfaces stable so they will not break as often after they have been released.

## Example import

This example can be used as a simple way to access the Code for IBM i instance.

```ts
import { CodeForIBMi } from "@halcyontech/vscode-ibmi-types";
import Instance from "@halcyontech/vscode-ibmi-types/api/Instance";
import { Extension, extensions } from "vscode";

let baseExtension: Extension<CodeForIBMi>|undefined;

/**
 * This should be used on your extension activation.
 */
export function loadBase(): CodeForIBMi|undefined {
  if (!baseExtension) {
    baseExtension = (extensions ? extensions.getExtension(`halcyontechltd.code-for-ibmi`) : undefined);
  }
  
  return (baseExtension && baseExtension.isActive && baseExtension.exports ? baseExtension.exports : undefined);
}

/**
 * Used when you want to fetch the extension 'instance' (the connection)
 */
export function getInstance(): Instance|undefined {
  return (baseExtension && baseExtension.isActive && baseExtension.exports ? baseExtension.exports.instance : undefined);
}
```

## Event listener

The Code for IBM i API provides an event listener. This allows your extension to fire an event when something happens in Code for IBM i.

```js
const instance = getInstance();

instance.onEvent(`connected`, () => {
  console.log(`It connected!`);
});
```

### Available events

| ID          | Event                                    |
|-------------|------------------------------------------|
| `connected` | When Code for IBM i connects to a system |
| `disconnected` | When Code for IBM i is disconnected from a system |
| `deployLocation` | When a deploy location changes |

# APIs

## Running commands with the user library list

Code for IBM i ships an API (via VS Code command) that can be used by an extension to execute a remote command on the IBM i.

It has a parameter which is an object with some properties:

```ts
interface CommandInfo {
  /** describes what environment the command will be executed. Is optional and defaults to `ile` */
  environment?: `pase`|`ile`|`qsh`;
  /** set this as the working directory for the command when it is executed. Is optional and defaults to the users working directory in Code for IBM i. */
  cwd?: string;
  command: string;
}
```

* Command can also use [Promptable fields](https://halcyon-tech.github.io/vscode-ibmi/#/?id=prompted).
* When executing a command in the `ile` or `qsh` environment, it will use the library list from the current connection.

The command returns an object:

```ts
interface CommandResult {
  stdout: string;
  stderr: string;
  code: number;
}
```

```js
const result: CommandResult = await vscode.commands.executeCommand(`code-for-ibmi.runCommand`, {
  environment: `pase`,
  command: `ls`
});

// or

const result = await vscode.commands.executeCommand(`code-for-ibmi.runCommand`, {
  environment: `pase`,
  command: `ls`
});
```

You can also provide a custom library list and current library when executing a command in the `ile` environment:

```js
const detail: CommandResult = {
  environment: `ile`,
  command: `CRTBNDRPG...`,
  env: {
    // Space delimited library list
    '&LIBL': 'LIBA LIBB LIBC'
    '&CURLIB': 'LIBD'
  }
}
```

## Running SQL queries

Code for IBM i has a command that lets you run SQL statements and get a result back.

```ts
const instance = getInstance();

const rows = await instance.getContent().runSQL(`select * from schema.yourtable`);
```

## Get members and streamfiles

It is possible for extensions to utilize the file systems provided by Code for IBM i.

`openTextDocument` returns a [`TextDocument`](https://code.visualstudio.com/api/references/vscode-api#TextDocument).

**Getting a member**

```js
const doc = await vscode.workspace.openTextDocument(vscode.Uri.from({
  scheme: `member`,
  path: `/${library}/${file}/${name}.${extension}`
}));
```

**Getting a streamfile**
```js
const doc = await vscode.workspace.openTextDocument(vscode.Uri.from({
  scheme: `streamfile`,
  path: streamfilePath
}));
```

## Connect to a system

It is possible for your API to automate connecting to an IBM i instead of the user using the connection view.

```ts
const connectionData: ConnectionData = {...};

const connected: boolean = await vscode.commands.executeCommand(`code-for-ibmi.connectDirect`, connectionData);

if (connected) {
  // do a thing.
} else {
  // something went wrong.
}
```

# VS Code integration

## Right click options

It is possible for your extension to add right click options to:

* objects in the Object Browser
* members in the Object Browser
* directories in the IFS Browser
* streamfiles in the IFS Browser
* much more

You would register a command as you'd normally expect, but expect a parameter for the chosen node from the tree view. Here is the sample for deleting a streamfile in the IFS Browser.

```js
context.subscriptions.push(
  // `node` is the object passed in directly from the IFS Browser.
  vscode.commands.registerCommand(`code-for-ibmi.deleteIFS`, async (node) => {
    if (node) {
      //Running from right click
      let result = await vscode.window.showWarningMessage(`Are you sure you want to delete ${node.path}?`, `Yes`, `Cancel`);

      if (result === `Yes`) {
        // directory using the connection API.
        const connection = instance.getConnection();

        try {
          // Run a pase command
          await vscode.commands.executeCommand(`code-for-ibmi.runCommand`, {
            command: `rm -rf "${node.path}`,
            environment: `pase`,
          });

          vscode.window.showInformationMessage(`Deleted ${node.path}.`);

          this.refresh();
        } catch (e) {
          vscode.window.showErrorMessage(`Error deleting streamfile! ${e}`);
        }
      }
    } else {
      // If it's reached this point, it usually 
      // indicates that it's running from the command palette
    }
  })
);
```

Following that, we need to register the command so it has a label. We do this in `package.json`

```json
{
  "command": "code-for-ibmi.deleteIFS",
  "title": "Delete object",
  "category": "Your extension"
}
```

Finally, we add it to a context menu:

```json
"menus": {
  "view/item/context": [
    {
      "command": "code-for-ibmi.deleteIFS",
      "when": "view == ifsBrowser",
      "group": "yourext@1"
    },
  ]
}
```

**When adding your command to a menu context**, there are lots of possible values for your `when` clause:

* `view` can be `ifsBrowser` or `objectBrowser`.
* `viewItem` can be different depending on the view:
   * for `ifsBrowser`, it can be `directory` or `streamfile`
   * for `objectBrowser`, it can be `member` (source member), `object` (any object), `SPF` (source file) or `filter`.

This allows your extension to provide commands for specific types of objects or specific items in the treeview.

[Read more about the when clause on the VS Code docs website.](https://code.visualstudio.com/api/references/when-clause-contexts)

## Views

Code for IBM i provides a context so you can control when a command, view, etc, can work. `code-for-ibmi.connected` can and should be used if your view depends on a connection. For example

This will show a welcome view when there is no connection:

```json
		"viewsWelcome": [{
			"view": "git-client-ibmi.commits",
			"contents": "No connection found. Please connect to an IBM i.",
			"when": "code-for-ibmi:connected !== true"
		}],
```

This will show a view when there is a connection:

```json
    "views": {
      "scm": [{
        "id": "git-client-ibmi.commits",
        "name": "Commits",
        "contextualTitle": "IBM i",
        "when": "code-for-ibmi:connected == true"
      }]
    }
```

# FAQs

## Getting the temporary library

Please remember that you cannot use `QTEMP` between commands since each command runs in a new job. Please refer to `instance.getConfig().tempLibrary` for the user temporary library.

## Is there a connection?

You can use `instance.getConnection()` to determine if there is a connection:

```ts
async getChildren(element) {
  const connection = instance.getConnection();

  let items: TreeItem[] = [];

  if (connection) {
    //Do work here...

  } else {
    items = [new vscode.TreeItem(`Please connect to an IBM i and refresh.`)];
  }

  return items;
}
```

## `connected` context

If you refer to the **Views** section, you can make it so the view is only shown when connected. This means by the time the view is used, there should be a connection.

```json
"views": {
  "explorer": [{
    "id": "yourIbmiView",
    "name": "My custom View",
    "contextualTitle": "Extension name",
    "when": "code-for-ibmi:connected == true"
  }]
}
```

```json
"activationEvents": [
    "onView:yourIbmiView"
]
```

## Examples

See the following code bases for large examples of extensions that use Code for IBM i:

* [VS Code extension to manage IBM i IWS services](https://github.com/halcyon-tech/vscode-ibmi-iws)
* [Git for IBM i extension](https://github.com/halcyon-tech/git-client-ibmi)
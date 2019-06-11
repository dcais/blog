---
title: 我的vscode配置
date: 2019-06-11 16:57:42
tags:
---

```json
// Place your settings in this file to overwrite the default settings
{
    "editor.tabSize": 2,
    "editor.renderWhitespace": "all",
    "editor.glyphMargin": true,
    "editor.wordWrap": "on",
    "eslint.enable": true,
    "java.home": "/Library/Java/JavaVirtualMachines/jdk1.8.0_111.jdk/Contents/Home",
    "workbench.iconTheme": "material-icon-theme",
    "vetur.validation.template": false,
    "files.autoSave": "onFocusChange",
    "window.zoomLevel": 0,
    "gitlens.advanced.messages": {
        "suppressShowKeyBindingsNotice": true
    },
    "gitlens.historyExplorer.enabled": true,
    "todo-tree.defaultHighlight": {
        "foreground": "green",
        "type": "none"
    },
    "todo-tree.customHighlight": {
        "TODO": {},
        "FIXME": {}
    },
    "eslint.alwaysShowStatus": true,
    "editor.formatOnType": false,
    "breadcrumbs.enabled": true,
    "search.quickOpen.includeSymbols": true,
    "gitlens.views.fileHistory.enabled": false,
    "gitlens.views.lineHistory.enabled": false,
    "terminal.integrated.shell.osx": "/usr/local/bin/zsh",
    "terminal.integrated.fontFamily": "Menlo for Powerline",
    "vs-kubernetes": {
        "vs-kubernetes.helm-path": "/Users/david/.vs-kubernetes/tools/helm/darwin-amd64/helm",
        "vs-kubernetes.draft-path": "/Users/david/.vs-kubernetes/tools/draft/darwin-amd64/draft"
    },
    "workbench.colorTheme": "One Dark Pro Bold",
    "material-icon-theme.folders.theme": "specific",
    "material-icon-theme.folders.color": "#26a69a",
    "material-icon-theme.activeIconPack": "angular",
    "editor.fontSize": 13,
    "terminal.integrated.fontSize": 14,
    "editor.fontFamily": "SourceCodePro-Medium,Menlo, Monaco, 'Courier New', monospace",
    "vetur.format.defaultFormatterOptions": {
        "js-beautify-html": {
            "wrap_attributes": "force-expand-multiline"
        },
        "prettyhtml": {
            "printWidth": 100,
            "singleQuote": false,
            "wrapAttributes": false,
            "sortAttributes": false
        },
        "prettier": {
            "semi": true,
            "singleQuote": true
        }
    },
    "eslint.autoFixOnSave": true,
    "files.associations": {
        "*.vue": "vue"
    },
    "eslint.validate": [
        "javascript",
        "javascriptreact",
        {
            "language": "vue",
            "autoFix": true
        }
    ],
    "java.configuration.checkProjectSettingsExclusions": false,
    "python.jediEnabled": false,
    "python.condaPath": "conda",
    "python.dataScience.sendSelectionToInteractiveWindow": true,
    "editor.suggestSelection": "first",
    "vsintellicode.modify.editor.suggestSelection": "automaticallyOverrodeDefaultValue",
    "yaml.format.enable": true,
    "diffEditor.ignoreTrimWhitespace": true,
    "java.jdt.ls.vmargs": "-noverify -Xmx1G -XX:+UseG1GC -XX:+UseStringDeduplication -javaagent:\"/Users/david/.vscode/extensions/gabrielbb.vscode-lombok-0.9.7/server/lombok.jar\" -Xbootclasspath/a:\"/Users/david/.vscode/extensions/gabrielbb.vscode-lombok-0.9.7/server/lombok.jar\"",
    "liveshare.featureSet": "insiders",
    "sync.gist": "xxxx",
    "sync.autoDownload": false,
    "sync.autoUpload": false,
    "sync.forceDownload": false,
    "sync.quietSync": false,
    "sync.askGistName": false,
    "sync.removeExtensions": true,
    "sync.syncExtensions": true
    // "eslint.options": {
    // "rules" : {
    // "vue/no-unused-vars":0 
    // }
    // }
}
```
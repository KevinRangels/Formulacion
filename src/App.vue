<template>
  <div>
    <div ref="editorContainer" class="editor-container"></div>
    <button @click="convertToPython">Convertir a Python</button>
    <pre>{{ pythonCode }}</pre> <!-- Mostrar el código Python generado -->
    <div v-if="errorMessage" class="error-message">{{ errorMessage }}</div> <!-- Mostrar el mensaje de error -->
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import * as monaco from 'monaco-editor';

const editorContainer = ref(null);
const pythonCode = ref('');
const errorMessage = ref(''); // Mensaje de error
let editor = null; // Aquí guardaremos la referencia al editor
let variables = {}; // Aquí almacenaremos las variables definidas por el usuario

onMounted(() => {
  editor = monaco.editor.create(editorContainer.value, {
    value: 'var1 = 10\nvar2 = 5\nSUMA(var1, var2)', // Contenido inicial con variables y fórmula
    language: 'plaintext',
    theme: 'vs-light',
    fontSize: 20, // Tamaño de fuente
    minimap: {
      enabled: false, // Desactiva el minimapa
    },
  });

  // Autocompletado personalizado con funciones básicas tipo Excel
  monaco.languages.registerCompletionItemProvider('plaintext', {
    provideCompletionItems: () => {
      const suggestions = [
        {
          label: 'SI',
          kind: monaco.languages.CompletionItemKind.Function,
          insertText: 'SI(${1:condicion}, ${2:valor_si_verdadero}, ${3:valor_si_falso})',
          insertTextRules: monaco.languages.CompletionItemInsertTextRule.InsertAsSnippet,
          documentation: 'Evalúa una condición y devuelve un valor si es verdadero y otro si es falso.'
        },
        {
          label: 'SUMA',
          kind: monaco.languages.CompletionItemKind.Function,
          insertText: 'SUMA(${1:valor1}, ${2:valor2}, ...)',
          insertTextRules: monaco.languages.CompletionItemInsertTextRule.InsertAsSnippet,
          documentation: 'Devuelve la suma de los valores proporcionados.',
          itemClassName: 'monaco-suggestion-item monaco-suggestion-variable',
        },
        {
          label: 'RESTAR',
          kind: monaco.languages.CompletionItemKind.Function,
          insertText: 'RESTAR(${1:valor1}, ${2:valor2})',
          insertTextRules: monaco.languages.CompletionItemInsertTextRule.InsertAsSnippet,
          documentation: 'Resta dos valores.'
        },
        {
          label: 'MULTIPLICAR',
          kind: monaco.languages.CompletionItemKind.Function,
          insertText: 'MULTIPLICAR(${1:valor1}, ${2:valor2})',
          insertTextRules: monaco.languages.CompletionItemInsertTextRule.InsertAsSnippet,
          documentation: 'Multiplica dos valores.'
        },
        {
          label: 'DIVIDIR',
          kind: monaco.languages.CompletionItemKind.Function,
          insertText: 'DIVIDIR(${1:valor1}, ${2:valor2})',
          insertTextRules: monaco.languages.CompletionItemInsertTextRule.InsertAsSnippet,
          documentation: 'Divide dos valores.'
        },
        {
          label: 'PROMEDIO',
          kind: monaco.languages.CompletionItemKind.Function,
          insertText: 'PROMEDIO(${1:valor1}, ${2:valor2}, ...)',
          insertTextRules: monaco.languages.CompletionItemInsertTextRule.InsertAsSnippet,
          documentation: 'Calcula el promedio de los valores proporcionados.'
        },
        {
          label: 'ENTRE',
          kind: monaco.languages.CompletionItemKind.Function,
          insertText: 'ENTRE(${1:valor}, ${2:min}, ${3:max})',
          insertTextRules: monaco.languages.CompletionItemInsertTextRule.InsertAsSnippet,
          documentation: 'Verifica si un valor está entre dos límites (inclusive).'
        },
        {
          label: 'verdadero',
          kind: monaco.languages.CompletionItemKind.Keyword,
          insertText: 'verdadero',
          documentation: 'Palabra clave: verdadero'
        },
        {
          label: 'falso',
          kind: monaco.languages.CompletionItemKind.Keyword,
          insertText: 'falso',
          documentation: 'Palabra clave: falso'
        }
      ];

      Object.keys(variables).forEach(varName => {
        suggestions.push({
          label: varName,
          kind: monaco.languages.CompletionItemKind.Variable,
          insertText: varName,
          documentation: `Variable definida: ${varName}`,
          itemClassName: 'monaco-suggestion-item monaco-suggestion-variable',
        });
      });

      return { suggestions: suggestions };
    }
  });

  // Resaltado de sintaxis para las funciones y palabras clave
  monaco.languages.setMonarchTokensProvider('plaintext', {
    tokenizer: {
      root: [
        [/\b(SI|SUMA|RESTAR|MULTIPLICAR|DIVIDIR|PROMEDIO|ENTRE|verdadero|falso)\b/, 'keyword'],
      ],
    },
  });

  // Validación y extracción de variables cada vez que cambia el contenido del editor
  editor.onDidChangeModelContent(() => {
    const formula = editor.getValue();
    validateAndExtractVariables(formula);
  });
});

// Función que convierte la fórmula ingresada a código Python
const convertToPython = () => {
  const formula = editor.getValue();
  pythonCode.value = convertFormulaToPython(formula);
};

// Validación y extracción de variables
const validateAndExtractVariables = (formula) => {
  // Limpiamos las variables previas
  variables = {};
  const lines = formula.split('\n');

  for (let line of lines) {
    const variableMatch = line.match(/(\w+)\s*=\s*(.+)/);
    if (variableMatch) {
      const [_, varName, varValue] = variableMatch;
      if (isNaN(varValue.trim())) {
        errorMessage.value = `Error en la asignación de la variable: "${line.trim()}"`;
        return;
      }
      variables[varName.trim()] = varValue.trim(); // Guardamos la variable
    }
  }

  errorMessage.value = ''; // No hay errores en la definición de variables
};

// Mapeo de fórmulas a código Python
const convertFormulaToPython = (formula) => {
  let pythonCode = formula;

  // Reemplazar variables en las fórmulas
  for (let [varName, varValue] of Object.entries(variables)) {
    pythonCode = pythonCode.replace(new RegExp(`\\b${varName}\\b`, 'g'), varValue);
  }

  return pythonCode
    .replace(/SI\((.+?),(.+?),(.+?)\)/g, '($1 if $2 else $3)')
    .replace(/SUMA\((.+?)\)/g, 'sum([$1])')
    .replace(/RESTAR\((.+?),(.+?)\)/g, '($1 - $2)')
    .replace(/MULTIPLICAR\((.+?),(.+?)\)/g, '($1 * $2)')
    .replace(/DIVIDIR\((.+?),(.+?)\)/g, '($1 / $2)')
    .replace(/PROMEDIO\((.+?)\)/g, '(sum([$1]) / len([$1]))')
    .replace(/ENTRE\((.+?),(.+?),(.+?)\)/g, '($2 <= $1 <= $3)')
    .replace(/verdadero/g, 'True')
    .replace(/falso/g, 'False');
};
</script>

<style scoped>
.editor-container {
  width: 100%;
  height: 300px;
  border: 2px solid #a855f7;
  border-radius: 8px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
}

.error-message {
  color: red;
  font-weight: bold;
}
/* Estilos para las sugerencias del autocompletado */
.monaco-suggestion-item {
  padding: 5px; /* Espaciado alrededor de cada elemento */
  border-radius: 4px; /* Bordes redondeados */
}

.monaco-suggestion-item:hover {
  background-color: #e6f7ff; /* Color de fondo al pasar el mouse */
}

.monaco-suggestion-function {
  font-weight: bold; /* Negrita para funciones */
  color: red; /* Color de texto para funciones */
}

.monaco-suggestion-variable {
  color: #008000; /* Color de texto para variables */
}
</style>

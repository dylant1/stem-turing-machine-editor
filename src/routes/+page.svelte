<script>
    import { onMount, onDestroy, tick } from 'svelte';
    import cytoscape from 'cytoscape';
    import cytoscapeCoseBilkent from 'cytoscape-cose-bilkent';
    import edgehandles from 'cytoscape-edgehandles';
    import undoRedo from 'cytoscape-undo-redo';
  
    let cy;
    let ur;
    let tmDefinition = ''; // Will be set after file upload
    let savedFunctions = []; // Array to store saved functions
    let functionCounter = 1; // Counter for function names
    let addTransitionMode = false;
    let transitionSourceNode = null;
  
    // Register the layout extension and edgehandles
    cytoscape.use(cytoscapeCoseBilkent);
    cytoscape.use(edgehandles);
    cytoscape.use(undoRedo);
  
    function handleFileUpload(event) {
      const file = event.target.files[0];
      if (file) {
        // Read the file content
        const reader = new FileReader();
        reader.onload = (e) => {
          tmDefinition = e.target.result;
          // Parse the content and render the graph
          parseAndRenderTMDefinition(tmDefinition);
        };
        reader.readAsText(file);
      }
    }
  
    function parseAndRenderTMDefinition(tmDefinition) {
      // Initialize arrays to store parsed states and transitions
      const tmStates = [];
      const tmTransitions = [];
  
      // Split the definition into lines and parse
      const lines = tmDefinition.split('\n');
      let currentSection = null;
  
      for (let line of lines) {
        line = line.trim();
        if (line.startsWith('//') || line === '') {
          continue; // Skip comments and empty lines
        }
        if (line === 'STATES:') {
          currentSection = 'STATES';
          continue;
        } else if (line === 'TRANSITION:') {
          currentSection = 'TRANSITION';
          continue;
        } else if (line === 'TAPE:') {
          currentSection = 'TAPE';
          continue;
        } else if (line.startsWith('Start Triangle Position:')) {
          currentSection = null;
          continue;
        }
  
        if (currentSection === 'STATES') {
          // Parse state line
          const tokens = line.split(/\s+/);
          const [id, x, y, start, accept] = tokens;
          tmStates.push({
            id: id,
            x: parseFloat(x),
            y: parseFloat(y),
            start: start === 'true',
            accept: accept === 'true',
          });
        } else if (currentSection === 'TRANSITION') {
          // Parse transition line
          const tokens = line.split(/\s+/);
          const [fromStateId, toStateId, readChar, writeChar, moveDirection] = tokens;
          tmTransitions.push({
            from: fromStateId,
            to: toStateId,
            readChar: readChar,
            writeChar: writeChar,
            moveDirection: moveDirection,
          });
        }
      }
  
      // Prepare elements for Cytoscape
      const elements = [];
  
      tmStates.forEach((state) => {
        elements.push({
          data: {
            id: state.id,
            label: state.id,
            start: state.start,
            accept: state.accept,
          },
          position: { x: state.x, y: state.y },
        });
      });
  
      tmTransitions.forEach((transition) => {
        elements.push({
          data: {
            id: `e${transition.from}-${transition.to}-${Math.random()}`,
            source: transition.from,
            target: transition.to,
            label: `${transition.readChar}→${transition.writeChar}, ${transition.moveDirection}`,
          },
        });
      });
  
      // Initialize Cytoscape
      initializeCytoscape(elements);
    }
  
    function initializeCytoscape(elements) {
      if (cy) {
        cy.destroy();
      }
  
      cy = cytoscape({
        container: document.getElementById('cy'), // Container to render in
        elements: elements,
        style: [
          {
            selector: 'node',
            style: {
              'background-color': '#fafad2', // Default background color
              'label': 'data(label)',
              'text-valign': 'center',
              'color': '#000',
              'width': '40px',
              'height': '40px',
              'shape': 'ellipse',
              'font-size': '12px',
              'border-width': 1,
              'border-color': '#000',
            },
          },
          {
            selector: 'edge',
            style: {
              'width': 2,
              'line-color': '#ccc',
              'target-arrow-color': '#ccc',
              'target-arrow-shape': 'triangle',
              'curve-style': 'bezier',
              'label': 'data(label)',
              'font-size': '10px',
              'text-rotation': 'autorotate',
              'text-margin-y': -10,
            },
          },
          {
            selector: 'node.start',
            style: {
              'border-width': 3,
              'border-color': 'green',
            },
          },
          {
            selector: 'node.accept',
            style: {
              'border-width': 3,
              'border-color': 'blue',
            },
          },
          {
            selector: 'node:selected',
            style: {
              'overlay-opacity': 0,
              'border-width': 3,
              'border-color': 'blue',
            },
          },
          {
            selector: 'edge:selected',
            style: {
              'overlay-opacity': 0,
              'line-color': 'blue',
              'target-arrow-color': 'blue',
            },
          },
        ],
        layout: {
          name: 'preset', // Use the positions specified in the elements
        },
        selectionType: 'single', // Enable single selection
        boxSelectionEnabled: true, // Enable box selection
      });
  
      // Initialize undo-redo extension
      ur = cy.undoRedo();
  
      // Apply classes for start and accept states
      cy.nodes().forEach((node) => {
        if (node.data('start')) {
          node.addClass('start');
        }
        if (node.data('accept')) {
          node.addClass('accept');
        }
      });
  
      // Enable pan and zoom
      cy.userZoomingEnabled(true);
      cy.userPanningEnabled(true);
  
      // Handle events
      cy.on('tap', (event) => {
        if (addTransitionMode && event.target.isNode()) {
          handleAddTransition(event.target);
          return;
        }
  
        if (event.target === cy) {
          if (event.originalEvent.detail === 2) {
            const { x, y } = event.position;
            addNodeAtPosition(x, y);
          }
        }
      });
  
      cy.on('cxttap', 'node', (event) => {
        const node = event.target;
        openNodeContextMenu(node, event.originalEvent);
      });
  
      cy.on('cxttap', 'edge', (event) => {
        const edge = event.target;
        openEdgeContextMenu(edge, event.originalEvent);
      });
  
      // Handle keydown events for duplicate, delete, undo, redo, and save function
      document.addEventListener('keydown', handleKeyDown);
  
      // Clean up event listeners when the component is destroyed
      onDestroy(() => {
        document.removeEventListener('keydown', handleKeyDown);
      });
  
      // Enable edge creation (we'll handle this manually)
    }
  
    function addNodeAtPosition(x, y) {
      const newId = getNextAvailableStateId();
      cy.add({
        group: 'nodes',
        data: { id: newId.toString(), label: newId.toString() },
        position: { x, y },
      });
    }
  
    function editEdgeLabel(edge) {
      const newLabel = prompt('Enter new label', edge.data('label'));
      if (newLabel !== null) {
        edge.data('label', newLabel);
      }
    }
  
    let contextMenuVisible = false;
    let contextMenuPosition = { x: 0, y: 0 };
    let contextNode = null;
    let contextEdge = null;
    let contextMenuType = ''; // 'node', 'edge'
  
    function openNodeContextMenu(node, event) {
      event.preventDefault();
      contextMenuVisible = true;
      contextMenuPosition = { x: event.clientX, y: event.clientY };
      contextNode = node;
      contextEdge = null;
      contextMenuType = 'node';
    }
  
    function openEdgeContextMenu(edge, event) {
      event.preventDefault();
      contextMenuVisible = true;
      contextMenuPosition = { x: event.clientX, y: event.clientY };
      contextEdge = edge;
      contextNode = null;
      contextMenuType = 'edge';
    }
  
    function closeContextMenu() {
      contextMenuVisible = false;
      contextNode = null;
      contextEdge = null;
      contextMenuType = '';
    }
  
    function toggleStartState() {
      ur.do('changeData', {
        ele: contextNode,
        data: { start: !contextNode.data('start') },
      });
      contextNode.toggleClass('start');
      closeContextMenu();
    }
  
    function toggleAcceptState() {
      ur.do('changeData', {
        ele: contextNode,
        data: { accept: !contextNode.data('accept') },
      });
      contextNode.toggleClass('accept');
      closeContextMenu();
    }
  
    function setNodeColor() {
      const color = prompt('Enter color (e.g., #FF0000)');
      if (color) {
        ur.do('changeStyle', {
          ele: contextNode,
          style: { 'background-color': color },
        });
      }
      closeContextMenu();
    }
  
    function editTransition() {
      const newLabel = prompt('Enter new transition label', contextEdge.data('label'));
      if (newLabel !== null) {
        ur.do('changeData', {
          ele: contextEdge,
          data: { label: newLabel },
        });
      }
      closeContextMenu();
    }
  
    // Function to get the next available state ID (smallest integer not in use)
    function getNextAvailableStateId() {
      const existingIds = cy.nodes().map((node) => parseInt(node.id())).filter((id) => !isNaN(id));
      let id = 0;
      while (existingIds.includes(id)) {
        id++;
      }
      return id;
    }
  
    // Handle keydown events for duplicate, delete, undo, redo, and save function
    function handleKeyDown(event) {
      if (event.metaKey || event.ctrlKey) {
        if (event.key === 'd') {
          event.preventDefault();
          duplicateSelection();
        } else if (event.key === 'z') {
          event.preventDefault();
          if (event.shiftKey) {
            ur.redo();
          } else {
            ur.undo();
          }
        } else if (event.key === 'f') {
          event.preventDefault();
          saveSelectionAsFunction();
        }
      } else if (event.key === 'Delete' || event.key === 'Backspace') {
        deleteSelection();
      }
    }
  
    function duplicateSelection() {
      const selectedNodes = cy.nodes(':selected');
      if (selectedNodes.length === 0) {
        return;
      }
  
      const selectedEdges = cy.edges().filter((edge) => {
        return selectedNodes.contains(edge.source()) && selectedNodes.contains(edge.target());
      });
  
      // Mapping of old IDs to new IDs
      const idMap = {};
  
      // Duplicate nodes
      ur.do('batch', () => {
        selectedNodes.forEach((node) => {
          const data = { ...node.data() };
          const newId = getNextAvailableStateId();
          idMap[node.id()] = newId.toString();
          data.id = newId.toString();
          data.label = newId.toString();
          const newNode = cy.add({
            group: 'nodes',
            data: data,
            position: {
              x: node.position('x'),
              y: node.position('y') + 50, // Place below the original
            },
          });
          if (node.hasClass('start')) {
            newNode.addClass('start');
          }
          if (node.hasClass('accept')) {
            newNode.addClass('accept');
          }
        });
  
        // Duplicate edges
        selectedEdges.forEach((edge) => {
          const data = { ...edge.data() };
          data.id = `e${idMap[data.source] || data.source}-${idMap[data.target] || data.target}-${Math.random()}`;
          data.source = idMap[data.source] || data.source;
          data.target = idMap[data.target] || data.target;
          cy.add({
            group: 'edges',
            data: data,
          });
        });
      });
  
      // Deselect all elements
      cy.elements().unselect();
  
      // Select new nodes
      const newNodes = cy.nodes().filter((node) => {
        return Object.values(idMap).includes(node.id());
      });
      newNodes.select();
    }
  
    function deleteSelection() {
      const selectedElements = cy.$(':selected');
      if (selectedElements.length > 0) {
        ur.do('remove', selectedElements);
      }
    }
  
    // Save selected nodes and edges as a reusable function
    async function saveSelectionAsFunction() {
      const selectedNodes = cy.nodes(':selected');
      if (selectedNodes.length === 0) {
        alert('No nodes selected to save as a function.');
        return;
      }
  
      const selectedEdges = cy.edges().filter((edge) => {
        return selectedNodes.contains(edge.source()) && selectedNodes.contains(edge.target());
      });
  
      const elementsJson = selectedNodes.union(selectedEdges).jsons();
  
      const functionName = 'Function ' + functionCounter++;
      const func = {
        name: functionName,
        elements: elementsJson,
        previewContainer: null, // This will be set by Svelte's bind:this
      };
      savedFunctions = [...savedFunctions, func];
  
      // Wait for the DOM to update
      await tick();
  
      // Initialize preview after Svelte updates the DOM
      initializeFunctionPreview(func);
    }
  
    // Initialize function previews
    function initializeFunctionPreview(func) {
      // Ensure the container is available
      if (!func.previewContainer) {
        console.error('Preview container not found');
        return;
      }
  
      const elements = func.elements;
      const previewCy = cytoscape({
        container: func.previewContainer,
        elements: elements,
        style: [
          {
            selector: 'node',
            style: {
              'background-color': '#fafad2',
              'label': 'data(label)',
              'text-valign': 'center',
              'color': '#000',
              'width': '20px',
              'height': '20px',
              'shape': 'ellipse',
              'font-size': '6px',
              'border-width': 1,
              'border-color': '#000',
            },
          },
          {
            selector: 'edge',
            style: {
              'width': 1,
              'line-color': '#ccc',
              'target-arrow-color': '#ccc',
              'target-arrow-shape': 'triangle',
              'curve-style': 'bezier',
              'label': 'data(label)',
              'font-size': '5px',
              'text-rotation': 'autorotate',
              'text-margin-y': -5,
            },
          },
          {
            selector: 'node.start',
            style: {
              'border-width': 2,
              'border-color': 'green',
            },
          },
          {
            selector: 'node.accept',
            style: {
              'border-width': 2,
              'border-color': 'blue',
            },
          },
        ],
        layout: {
          name: 'cose-bilkent',
          fit: true,
          animate: false,
        },
        userZoomingEnabled: false,
        userPanningEnabled: false,
        boxSelectionEnabled: false,
      });
  
      // Apply classes for start and accept states
      previewCy.nodes().forEach((node) => {
        if (node.data('start')) {
          node.addClass('start');
        }
        if (node.data('accept')) {
          node.addClass('accept');
        }
      });
  
      // Disable interaction with the preview
      previewCy.on('tap', (event) => {
        event.preventDefault();
      });
  
      // Implement drag and drop from preview to main canvas
      func.previewContainer.addEventListener('dragstart', (e) => {
        e.dataTransfer.setData('functionIndex', savedFunctions.indexOf(func));
      });
  
      // Set draggable attribute
      func.previewContainer.setAttribute('draggable', 'true');
    }
  
    // Handle drop event on main canvas
    function handleDrop(event) {
      event.preventDefault();
      const functionIndex = event.dataTransfer.getData('functionIndex');
      if (functionIndex !== null) {
        const func = savedFunctions[functionIndex];
        const rect = cy.container().getBoundingClientRect();
        const x = event.clientX - rect.left;
        const y = event.clientY - rect.top;
        const position = cy.renderer().projectIntoViewport(x, y);
  
        insertFunction(func, position);
      }
    }
  
    function handleDragOver(event) {
      event.preventDefault();
    }
  
    // Insert a saved function into the graph at a position
    function insertFunction(func, position) {
      ur.do('batch', () => {
        const idMap = {};
        const existingIds = cy.nodes().map((node) => node.id());
        const elementsToAdd = func.elements.map((ele) => {
          const data = { ...ele.data };
          if (ele.group === 'nodes') {
            const newId = getNextAvailableStateId();
            idMap[data.id] = newId.toString();
            data.id = newId.toString();
            data.label = newId.toString();
            ele.data = data;
            ele.position = {
              x: ele.position.x + position.x - 50,
              y: ele.position.y + position.y - 50,
            };
          } else if (ele.group === 'edges') {
            data.source = idMap[data.source];
            data.target = idMap[data.target];
            data.id = `e${data.source}-${data.target}-${Math.random()}`;
            ele.data = data;
          }
          return ele;
        });
  
        cy.add(elementsToAdd);
      });
    }
  
    // Function to export the Turing machine definition
    function exportTMDefinition() {
      let output = '';
      output += 'STATES:\n';
      cy.nodes().forEach((node) => {
        const id = node.id();
        const x = node.position('x').toFixed(2);
        const y = node.position('y').toFixed(2);
        const start = node.data('start') ? 'true' : 'false';
        const accept = node.data('accept') ? 'true' : 'false';
        output += `${id} ${x} ${y} ${start} ${accept}\n`;
      });
  
      output += 'TRANSITION:\n';
      cy.edges().forEach((edge) => {
        const source = edge.source().id();
        const target = edge.target().id();
        const label = edge.data('label');
        // Assuming the label is in the format 'readChar→writeChar, moveDirection'
        const [readWrite, moveDirection] = label.split(',').map((s) => s.trim());
        const [readChar, writeChar] = readWrite.split('→').map((s) => s.trim());
        output += `${source} ${target} ${readChar} ${writeChar} ${moveDirection}\n`;
      });
  
      // For simplicity, we won't include the TAPE and Start Triangle Position sections
      downloadTextFile('turing_machine_definition.txt', output);
    }
  
    function downloadTextFile(filename, content) {
      const element = document.createElement('a');
      element.setAttribute(
        'href',
        'data:text/plain;charset=utf-8,' + encodeURIComponent(content)
      );
      element.setAttribute('download', filename);
      element.style.display = 'none';
      document.body.appendChild(element);
      element.click();
      document.body.removeChild(element);
    }
  
    // Add Transition functionality
    function startAddTransition() {
      addTransitionMode = true;
      transitionSourceNode = null;
      alert('Add Transition Mode: Click on the source node, then the target node.');
    }
  
    function handleAddTransition(node) {
      if (!transitionSourceNode) {
        transitionSourceNode = node;
      } else {
        const sourceNode = transitionSourceNode;
        const targetNode = node;
        const label = prompt('Enter transition label (format: readChar→writeChar, moveDirection)');
        if (label !== null) {
          ur.do('add', {
            group: 'edges',
            data: {
              id: `e${sourceNode.id()}-${targetNode.id()}-${Math.random()}`,
              source: sourceNode.id(),
              target: targetNode.id(),
              label: label,
            },
          });
        }
        addTransitionMode = false;
        transitionSourceNode = null;
      }
    }
  
    // Custom action to detect clicks outside an element
    function clickOutside(node) {
      const handleClick = (event) => {
        if (!node.contains(event.target)) {
          closeContextMenu();
        }
      };
  
      document.addEventListener('click', handleClick);
  
      return {
        destroy() {
          document.removeEventListener('click', handleClick);
        },
      };
    }
  
    // Initialize Cytoscape when the component mounts
    onMount(() => {
      initializeCytoscape([]);
  
      // Add event listeners for drag and drop
      const cyContainer = document.getElementById('cy');
      cyContainer.addEventListener('dragover', handleDragOver);
      cyContainer.addEventListener('drop', handleDrop);
    });
  </script>
  
  <div>
    <input type="file" on:change={handleFileUpload} />
    <button on:click={exportTMDefinition}>Export Turing Machine</button>
    <button on:click={startAddTransition}>Add Transition</button>
  </div>
  
  <div id="cy" style="width: 100%; height: 70vh; border: 1px solid #ccc;"></div>
  
  {#if contextMenuVisible}
    {#if contextMenuType === 'node'}
      <div
        class="context-menu"
        style="top: {contextMenuPosition.y}px; left: {contextMenuPosition.x}px;"
        use:clickOutside
      >
        <ul>
          <li on:click={toggleStartState}>Toggle Start State</li>
          <li on:click={toggleAcceptState}>Toggle Accept State</li>
          <li on:click={setNodeColor}>Set Color</li>
        </ul>
      </div>
    {:else if contextMenuType === 'edge'}
      <div
        class="context-menu"
        style="top: {contextMenuPosition.y}px; left: {contextMenuPosition.x}px;"
        use:clickOutside
      >
        <ul>
          <li on:click={editTransition}>Edit Transition</li>
        </ul>
      </div>
    {/if}
  {/if}
  
  <div>
    <h3>Saved Functions</h3>
    {#if savedFunctions.length > 0}
      <ul>
        {#each savedFunctions as func}
          <li>
            {func.name}
            <div
              class="function-preview"
              bind:this={func.previewContainer}
              on:dragstart
            ></div>
          </li>
        {/each}
      </ul>
    {:else}
      <p>No saved functions.</p>
    {/if}
  </div>
  
  <style>
    .context-menu {
      position: absolute;
      background-color: white;
      border: 1px solid #ccc;
      z-index: 1000;
    }
  
    .context-menu ul {
      list-style-type: none;
      margin: 0;
      padding: 0;
    }
  
    .context-menu li {
      padding: 8px 12px;
      cursor: pointer;
    }
  
    .context-menu li:hover {
      background-color: #eee;
    }
  
    /* Edgehandles styles */
    .eh-handle {
      background-color: #666;
      width: 12px;
      height: 12px;
    }
  
    .eh-hover {
      background-color: #aaa;
    }
  
    .eh-source {
      border-color: #333;
    }
  
    .eh-target {
      border-color: #333;
    }
  
    .function-preview {
      width: 100px;
      height: 100px;
      border: 1px solid #ccc;
      display: inline-block;
      margin-left: 10px;
    }
  
    .function-preview[draggable="true"] {
      cursor: move;
    }
  </style>
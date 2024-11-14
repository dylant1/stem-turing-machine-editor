<script>
    import { onMount, onDestroy, tick } from 'svelte';
    import cytoscape from 'cytoscape';
    import cytoscapeCoseBilkent from 'cytoscape-cose-bilkent';
    import undoRedo from 'cytoscape-undo-redo';
  
    let cy;
    let ur;
    let tmDefinition = ''; // Will be set after file upload
    let savedFunctions = []; // Array to store saved functions
    let functionCounter = 1; // Counter for function names
    let addTransitionMode = false;
    let transitionSourceNode = null;
  
    let showTransitionModal = false;
    let transitionReadChar = '';
    let transitionWriteChar = '';
    let transitionMoveDirection = 'RIGHT'; // Default direction
    let transitionCallback = null;
  
    let showEdgeLabelModal = false;
    let edgeToEdit = null;
    let edgeLabel = '';
  
    // Global counter for node IDs
    let stateIdCounter = 0;
  
    // Variable to store last mouse position
    let lastMousePosition = { x: 0, y: 0 };
  
    // Variables for context menu
    let contextMenuVisible = false;
    let contextMenuPosition = { x: 0, y: 0 };
    let contextNode = null;
    let contextEdge = null;
    let contextMenuType = ''; // 'node', 'edge'
  
    // Tape simulation variables (if applicable)
    let tape = [];
    let tapeHeadPosition = 0;
    let currentState = null;
    let isAcceptState = false;
    let isRunning = false;
    let simulationInterval = null;
    let transitionMap = {};
  
    // Register the layout extension and undo-redo
    cytoscape.use(cytoscapeCoseBilkent);
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
      let tapeLines = [];
      tapeHeadPosition = 0;
  
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
          // Extract the tapeHeadPosition
          const tokens = line.split(':');
          tapeHeadPosition = parseInt(tokens[1].trim());
          continue;
        }
  
        if (currentSection === 'STATES') {
          // Parse state line
          const tokens = line.split(/\s+/);
          const [id, x, y, start, accept, r, g, b, a] = tokens;
          // Convert RGBA values to CSS color string
          const color = `rgba(${parseFloat(r) * 255}, ${parseFloat(g) * 255}, ${parseFloat(b) * 255}, ${parseFloat(a)})`;
          tmStates.push({
            id: id,
            x: parseFloat(x),
            y: parseFloat(y),
            start: start === 'true',
            accept: accept === 'true',
            color: color,
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
        } else if (currentSection === 'TAPE') {
          if (!line.startsWith('Start Triangle Position:') && !line.startsWith('//') && line !== '') {
            if (isNaN(parseInt(line))) {
              tapeLines.push(line);
            }
          }
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
            color: state.color, // Include the color data
          },
          position: { x: state.x, y: state.y },
        });
      });
  
      tmTransitions.forEach((transition) => {
        const transitionLabel = `${transition.readChar};${transition.writeChar};${transition.moveDirection}`;
  
        // Check if an edge between from and to already exists
        let edge = elements.find(
          (ele) =>
            ele.group === 'edges' &&
            ele.data.source === transition.from &&
            ele.data.target === transition.to
        );
  
        if (edge) {
          // Edge exists, append the transition label
          edge.data.label += `\n${transitionLabel}`;
        } else {
          // Edge does not exist, create a new one
          elements.push({
            data: {
              id: `e${transition.from}-${transition.to}-${Math.random()}`,
              source: transition.from,
              target: transition.to,
              label: transitionLabel,
            },
          });
        }
      });
  
      // Initialize Cytoscape
      initializeCytoscape(elements);
  
      // Initialize the tape
      tape = tapeLines.join('').split('');
      // Find the start state
      const startStates = tmStates.filter((state) => state.start);
      if (startStates.length > 0) {
        currentState = startStates[0].id;
      } else {
        // If no start state, use the first state
        currentState = tmStates[0].id;
      }
      isAcceptState = false;
  
      // Build the transition map
      transitionMap = {};
  
      tmTransitions.forEach((transition) => {
        const fromState = transition.from;
        const readChar = transition.readChar;
        if (!transitionMap[fromState]) {
          transitionMap[fromState] = {};
        }
        if (readChar === '~') {
          transitionMap[fromState]['~'] = transition;
        } else {
          transitionMap[fromState][readChar] = transition;
        }
      });
  
      // Update visualization
      updateVisualization();
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
              'background-color': 'data(color)', // Use color from data
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
            selector: 'node.current',
            style: {
              'border-color': 'red',
              'border-width': 3,
            },
          },
          {
            selector: 'node:selected',
            style: {
              'overlay-opacity': 0.2, // Set to a visible opacity
              'overlay-color': '#dfeffc', // Optional: Set overlay color
              'border-width': 3,
              'border-color': 'blue',
            },
          },
          {
            selector: 'edge:selected',
            style: {
              'overlay-opacity': 0.2,
              'overlay-color': '#dfeffc',
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
        autounselectify: false, // Ensure selection is enabled
      });
  
      // Enable user interactions
      cy.userZoomingEnabled(true);
      cy.userPanningEnabled(true);
  
      // Initialize undo-redo extension
      ur = cy.undoRedo();
  
      // Register 'changeData' action
      ur.action(
        'changeData',
        function (args) {
          const ele = args.ele;
          const data = args.data;
          const oldData = {};
  
          for (let key in data) {
            oldData[key] = ele.data(key);
            ele.data(key, data[key]);
          }
  
          return { ele: ele, data: oldData };
        },
        function (args) {
          const ele = args.ele;
          const data = args.data;
  
          for (let key in data) {
            ele.data(key, data[key]);
          }
        }
      );
  
      // Set stateIdCounter to the maximum numeric node ID plus one
      const numericIds = cy.nodes().map((node) => parseInt(node.id())).filter((id) => !isNaN(id));
      if (numericIds.length > 0) {
        stateIdCounter = Math.max(...numericIds) + 1;
      } else {
        stateIdCounter = 0;
      }
  
      // Apply classes for start and accept states
      cy.nodes().forEach((node) => {
        if (node.data('start')) {
          node.addClass('start');
        }
        if (node.data('accept')) {
          node.addClass('accept');
        }
      });
  
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
  
      // Event listener to check if nodes are selectable
      cy.on('select', 'node', (event) => {
        console.log('Node selected:', event.target.id());
      });
  
      // Clean up event listeners when the component is destroyed
    //   onDestroy(() => {
    //     document.removeEventListener('keydown', handleKeyDown);
    //   });
    }
  
    function addNodeAtPosition(x, y) {
      const newId = getNextAvailableStateId().toString();
      cy.add({
        group: 'nodes',
        data: { id: newId, label: newId, color: '#fafad2' },
        position: { x, y },
      });
    }
  
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
      const color = prompt('Enter color (e.g., #FF0000 or rgba(255,0,0,1))');
      if (color) {
        let nodesToChange = cy.nodes(':selected');
        if (nodesToChange.length === 0 && contextNode) {
          nodesToChange = cy.collection([contextNode]);
        }
        nodesToChange.forEach((node) => {
          ur.do('changeData', {
            ele: node,
            data: { color: color },
          });
        });
      }
      closeContextMenu();
    }
  
    function editEdgeLabel(edge) {
      edgeToEdit = edge;
      edgeLabel = edge.data('label');
      showEdgeLabelModal = true;
      closeContextMenu();
    }
  
    function saveEdgeLabel() {
      if (edgeToEdit && edgeLabel !== null) {
        ur.do('changeData', {
          ele: edgeToEdit,
          data: { label: edgeLabel },
        });
      }
      showEdgeLabelModal = false;
      edgeToEdit = null;
      edgeLabel = '';
    }
  
    function cancelEdgeLabelEdit() {
      showEdgeLabelModal = false;
      edgeToEdit = null;
      edgeLabel = '';
    }
  
    // Function to get the next available state ID (unique and numeric)
    function getNextAvailableStateId() {
      while (cy.getElementById(stateIdCounter.toString()).length > 0) {
        stateIdCounter++;
      }
      return stateIdCounter++;
    }
  
    // Handle keydown events for duplicate, delete, undo, redo, and save function
    function handleKeyDown(event) {
      if (event.metaKey || event.ctrlKey) {
        const key = event.key.toLowerCase();
        console.log(`Key pressed with Ctrl/Cmd: ${key}`);
        if (key === 'd') {
          event.preventDefault();
          console.log('Duplicate command detected');
          duplicateSelection();
        } else if (key === 'z') {
          event.preventDefault();
          if (event.shiftKey) {
            ur.redo();
            console.log('Redo action performed');
          } else {
            ur.undo();
            console.log('Undo action performed');
          }
        } else if (key === 'f') {
          event.preventDefault();
          console.log('Save function command detected');
          saveSelectionAsFunction();
        }
      } else if (event.key === 'Delete' || event.key === 'Backspace') {
        deleteSelection();
        console.log('Delete action performed');
      }
    }
  
    function duplicateSelection() {
      const selectedNodes = cy.nodes(':selected');
  
      console.log(`Selected nodes for duplication: ${selectedNodes.map((n) => n.id()).join(', ')}`);
      if (selectedNodes.length === 0) {
        console.log('No nodes selected to duplicate');
        return;
      }
  
      const selectedEdges = cy.edges().filter((edge) => {
        return selectedNodes.contains(edge.source()) && selectedNodes.contains(edge.target());
      });
      console.log('Selected edges for duplication:', selectedEdges.map((e) => e.id()));
  
      // Mapping of old IDs to new IDs
      const idMap = {};
  
      const elementsToAdd = [];
  
      selectedNodes.forEach((node) => {
        const data = { ...node.data() };
        const newId = getNextAvailableStateId().toString();
        idMap[node.id()] = newId;
        data.id = newId;
        data.label = newId;
        // Preserve 'start', 'accept', and 'color' properties
        data.start = node.data('start');
        data.accept = node.data('accept');
        data.color = node.data('color'); // Preserve color
        const position = {
          x: node.position('x') + 50,
          y: node.position('y') + 50, // Place diagonally below the original
        };
        const newNode = {
          group: 'nodes',
          data: data,
          position: position,
        };
        elementsToAdd.push(newNode);
        console.log(`Duplicated node ${node.id()} as ${newId}`);
      });
  
      selectedEdges.forEach((edge) => {
        const data = { ...edge.data() };
        data.source = idMap[data.source] || data.source;
        data.target = idMap[data.target] || data.target;
        data.id = `e${data.source}-${data.target}-${Math.random()}`;
        const newEdge = {
          group: 'edges',
          data: data,
        };
        elementsToAdd.push(newEdge);
        console.log(`Duplicated edge from ${data.source} to ${data.target}`);
      });
  
      // Add all elements using ur.do('add', elementsToAdd)
      ur.do('add', elementsToAdd);
  
      // Deselect all elements
      cy.elements().unselect();
  
      // Select new nodes
      const newNodes = cy.nodes().filter((node) => {
        return Object.values(idMap).includes(node.id());
      });
  
      newNodes.select();
      console.log(`New nodes selected after duplication: ${newNodes.map((n) => n.id()).join(', ')}`);
    }
  
    function deleteSelection() {
      const selectedElements = cy.$(':selected');
      if (selectedElements.length > 0) {
        ur.do('remove', selectedElements);
        console.log(`Deleted elements: ${selectedElements.map((ele) => ele.id()).join(', ')}`);
      } else {
        console.log('No elements selected to delete');
      }
    }
  
    // Save selected nodes and edges as a reusable function
    async function saveSelectionAsFunction() {
      const selectedNodes = cy.nodes(':selected');
      console.log(`Selected nodes for function: ${selectedNodes.map((n) => n.id()).join(', ')}`);
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
      console.log(`Saved new function: ${functionName}`);
  
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
              'background-color': 'data(color)',
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
        autoungrabify: true, // Disable dragging nodes
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
    }
  
    // Handle drag start for function previews
    function handleFunctionDragStart(func) {
      return (e) => {
        console.log(`Function drag started: ${func.name}`);
        e.dataTransfer.setData('functionIndex', savedFunctions.indexOf(func));
      };
    }
  
    // Handle drop event on main canvas
    function handleDrop(event) {
      event.preventDefault();
      const functionIndex = event.dataTransfer.getData('functionIndex');
      console.log(`Function dropped, index: ${functionIndex}`);
      if (functionIndex !== null && functionIndex !== '') {
        const func = savedFunctions[functionIndex];
        const rect = cy.container().getBoundingClientRect();
  
        // Use lastMousePosition
        const x = lastMousePosition.x - rect.left;
        const y = lastMousePosition.y - rect.top;
  
        // Convert screen coordinates to model coordinates
        const position = convertScreenToModelPosition(x, y);
  
        console.log(`Inserting function "${func.name}" at position (${position.x}, ${position.y})`);
        insertFunction(func, position);
      }
    }
  
    function handleDragOver(event) {
      event.preventDefault();
      lastMousePosition = {
        x: event.clientX,
        y: event.clientY,
      };
    }
  
    function convertScreenToModelPosition(x, y) {
      const pan = cy.pan();
      const zoom = cy.zoom();
  
      const modelPosition = {
        x: (x - pan.x) / zoom,
        y: (y - pan.y) / zoom,
      };
  
      return modelPosition;
    }
  
    // Insert a saved function into the graph at a position
    function insertFunction(func, position) {
      console.log('Inserting function');
  
      const idMap = {};
  
      const funcNodes = func.elements.filter((ele) => ele.group === 'nodes');
  
      if (funcNodes.length === 0) return;
  
      // Calculate centroid of the function nodes
      const centroid = {
        x: funcNodes.reduce((sum, node) => sum + node.position.x, 0) / funcNodes.length,
        y: funcNodes.reduce((sum, node) => sum + node.position.y, 0) / funcNodes.length,
      };
  
      const offset = {
        x: position.x - centroid.x,
        y: position.y - centroid.y,
      };
  
      console.log(`Function centroid: (${centroid.x}, ${centroid.y}), offset: (${offset.x}, ${offset.y})`);
  
      const elementsToAdd = func.elements.map((ele) => {
        const data = { ...ele.data };
        if (ele.group === 'nodes') {
          const newId = getNextAvailableStateId().toString();
          idMap[data.id] = newId;
          data.id = newId;
          data.label = newId;
          // Preserve 'start', 'accept', and 'color' properties
          data.start = ele.data.start;
          data.accept = ele.data.accept;
          data.color = ele.data.color;
          ele.data = data;
          ele.position = {
            x: ele.position.x + offset.x,
            y: ele.position.y + offset.y,
          };
          console.log(`Adding node ${data.id} at position (${ele.position.x}, ${ele.position.y})`);
        } else if (ele.group === 'edges') {
          data.source = idMap[data.source];
          data.target = idMap[data.target];
          data.id = `e${data.source}-${data.target}-${Math.random()}`;
          ele.data = data;
          console.log(`Adding edge from ${data.source} to ${data.target}`);
        }
        return ele;
      });
  
      // Add all elements using ur.do('add', elementsToAdd)
      ur.do('add', elementsToAdd);
    }
  
    // Function to parse color strings into RGBA fractions
    function parseColorString(colorStr) {
      let r = 0.98,
        g = 0.98,
        b = 0.82,
        a = 1; // Default values
      if (colorStr.startsWith('rgba')) {
        const rgbaMatch = colorStr.match(/rgba\(\s*(\d+),\s*(\d+),\s*(\d+),\s*(\d*(?:\.\d+)?)\s*\)/);
        if (rgbaMatch) {
          r = parseInt(rgbaMatch[1]) / 255;
          g = parseInt(rgbaMatch[2]) / 255;
          b = parseInt(rgbaMatch[3]) / 255;
          a = parseFloat(rgbaMatch[4]);
        }
      } else if (colorStr.startsWith('rgb')) {
        const rgbMatch = colorStr.match(/rgb\(\s*(\d+),\s*(\d+),\s*(\d+)\s*\)/);
        if (rgbMatch) {
          r = parseInt(rgbMatch[1]) / 255;
          g = parseInt(rgbMatch[2]) / 255;
          b = parseInt(rgbMatch[3]) / 255;
          a = 1;
        }
      } else if (colorStr.startsWith('#')) {
        if (colorStr.length === 7) {
          r = parseInt(colorStr.substr(1, 2), 16) / 255;
          g = parseInt(colorStr.substr(3, 2), 16) / 255;
          b = parseInt(colorStr.substr(5, 2), 16) / 255;
          a = 1;
        }
      }
      return { r, g, b, a };
    }
  
    // Function to export the Turing machine definition
    function exportTMDefinition() {
      let output = '';
      output += '// Save File for STEM\n';
      output += '// Version 1.00\n\n';
  
      // State Format: name x y start accept
      output += '// State Format: name x y start accept\n';
      output += 'STATES:\n';
  
      // Get the height of the Cytoscape container
      const containerHeight = cy.container().offsetHeight;
  
      cy.nodes().forEach((node) => {
        const id = node.id();
        const x = node.position('x');
        const y = node.position('y');
  
        // Invert the y-coordinate
        const invertedY = Math.abs(y);
  
        const start = node.data('start') ? 'true' : 'false';
        const accept = node.data('accept') ? 'true' : 'false';
        // Get color data
        const colorStr = node.data('color') || '#fafad2'; // Default color
        const rgba = parseColorString(colorStr);
        output += `\t${id} ${x} ${invertedY} ${start} ${accept} ${rgba.r} ${rgba.g} ${rgba.b} ${rgba.a}\n`;
      });
  
      // Transition format: fromStateId toStateId readChar writeChar moveDirection
      // The Character '~' is the catchall character
      output += '\n// Transition format: fromStateId toStateId readChar writeChar moveDirection\n';
      output += "// The Character '~' is the catchall character\n";
      output += 'TRANSITION:\n';
  
      cy.edges().forEach((edge) => {
        const source = edge.source().id();
        const target = edge.target().id();
        const labels = edge.data('label').split('\n');
        labels.forEach((label) => {
          const [readChar, writeChar, moveDirection] = label.split(';').map((s) => s.trim());
          output += `\t${source} ${target} ${readChar} ${writeChar} ${moveDirection}\n`;
        });
      });
  
      // Tape format: tapeChar(0) tapeChar(1) ... tapeChar(n)
      output += '\n// Tape format: tapeChar(0) tapeChar(1) ... tapeChar(n)\n';
      output += 'TAPE:\n';
      output += `\t${tapeHeadPosition}\n`; // Include the number before the tape content
      output += `\t${tape.join('')}\n`;
      output += `Start Triangle Position:${tapeHeadPosition}\n`;
  
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
      console.log('Add Transition Mode: Click on the source node, then the target node.');
    }
  
    function handleAddTransition(node) {
      if (!transitionSourceNode) {
        transitionSourceNode = node;
        console.log(`Transition source node selected: ${node.id()}`);
      } else {
        const sourceNode = transitionSourceNode;
        const targetNode = node;
        transitionSourceNode = null;
        addTransitionMode = false;
  
        console.log(`Transition target node selected: ${node.id()}`);
  
        // Show modal to enter transition details
        showTransitionModal = true;
  
        transitionCallback = (readChar, writeChar, moveDirection) => {
          if (readChar && writeChar && moveDirection) {
            addOrUpdateEdge(sourceNode.id(), targetNode.id(), readChar, writeChar, moveDirection);
          } else {
            console.log('Transition input cancelled or incomplete');
          }
          showTransitionModal = false;
          transitionReadChar = '';
          transitionWriteChar = '';
          transitionMoveDirection = 'RIGHT';
          transitionCallback = null;
        };
      }
    }
  
    function addOrUpdateEdge(sourceId, targetId, readChar, writeChar, moveDirection) {
      // Check if an edge already exists between source and target
      let edge = cy
        .edges()
        .filter((edge) => edge.source().id() === sourceId && edge.target().id() === targetId)
        .first();
  
      const transitionLabel = `${readChar};${writeChar};${moveDirection}`;
  
      if (edge && edge.length > 0) {
        // Edge exists, update the label
        let existingLabel = edge.data('label');
        const labels = existingLabel ? existingLabel.split('\n') : [];
        labels.push(transitionLabel);
        const newLabel = labels.join('\n');
        ur.do('changeData', {
          ele: edge,
          data: { label: newLabel },
        });
        console.log(`Updated edge between ${sourceId} and ${targetId} with new transition`);
      } else {
        // Edge does not exist, create a new one
        ur.do('add', {
          group: 'edges',
          data: {
            id: `e${sourceId}-${targetId}-${Math.random()}`,
            source: sourceId,
            target: targetId,
            label: transitionLabel,
          },
        });
        console.log(`Added new edge from ${sourceId} to ${targetId} with transition`);
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
      console.log('Cytoscape initialized and event listeners added');
    });
  
    // Update visualization function (if applicable)
    function updateVisualization() {
      // Implement this function if you have visualization updates
    }
  
    // Tape simulation functions (simulateStep, etc.) if needed
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
          <li on:click={() => editEdgeLabel(contextEdge)}>Edit Transition</li>
        </ul>
      </div>
    {/if}
  {/if}
  
  {#if showTransitionModal}
    <div class="modal-overlay">
      <div class="modal">
        <h3>Enter Transition Details</h3>
        <label>
          Read Character:
          <input type="text" bind:value={transitionReadChar} />
        </label>
        <label>
          Write Character:
          <input type="text" bind:value={transitionWriteChar} />
        </label>
        <label>
          Move Direction:
          <select bind:value={transitionMoveDirection}>
            <option value="LEFT">Left</option>
            <option value="RIGHT">Right</option>
            <option value="STAY">Stay</option>
          </select>
        </label>
        <button
          on:click={() =>
            transitionCallback(transitionReadChar, transitionWriteChar, transitionMoveDirection)
          }
        >
          OK
        </button>
        <button
          on:click={() => {
            showTransitionModal = false;
            transitionReadChar = '';
            transitionWriteChar = '';
            transitionMoveDirection = 'RIGHT';
            transitionCallback = null;
          }}
        >
          Cancel
        </button>
      </div>
    </div>
  {/if}
  
  {#if showEdgeLabelModal}
    <div class="modal-overlay">
      <div class="modal">
        <h3>Edit Transition Label</h3>
        <textarea bind:value={edgeLabel} rows="5" style="width: 100%;"></textarea>
        <button on:click={saveEdgeLabel}>Save</button>
        <button on:click={cancelEdgeLabelEdit}>Cancel</button>
      </div>
    </div>
  {/if}
  
  <div>
    <h3>Saved Functions</h3>
    {#if savedFunctions.length > 0}
      <ul>
        {#each savedFunctions as func}
          <li>
            <div
              class="function-preview-container"
              draggable="true"
              on:dragstart={handleFunctionDragStart(func)}
            >
              {func.name}
              <div
                class="function-preview"
                bind:this={func.previewContainer}
              ></div>
            </div>
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
  
    .function-preview-container {
      display: inline-block;
      cursor: move;
      margin: 10px;
      user-select: none;
    }
  
    .function-preview {
      width: 100px;
      height: 100px;
      border: 1px solid #ccc;
      pointer-events: none; /* Disable interactions */
    }
  
    .modal-overlay {
      position: fixed;
      top: 0;
      left: 0;
      right: 0;
      bottom: 0;
      background: rgba(0, 0, 0, 0.5);
      display: flex;
      justify-content: center;
      align-items: center;
      z-index: 2000;
    }
  
    .modal {
      background: white;
      padding: 20px;
      border-radius: 4px;
      min-width: 300px;
    }
  
    .modal h3 {
      margin-top: 0;
    }
  
    .modal input,
    .modal textarea,
    .modal select {
      width: 100%;
      padding: 8px;
      margin-bottom: 10px;
    }
  
    .modal button {
      padding: 8px 12px;
      margin-right: 10px;
    }
  </style>
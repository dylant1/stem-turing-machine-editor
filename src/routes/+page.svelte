<script>
  import { onMount, tick } from 'svelte';
  import cytoscape from 'cytoscape';
  import cytoscapeCoseBilkent from 'cytoscape-cose-bilkent';
  import undoRedo from 'cytoscape-undo-redo';

  let cy;
  let ur;
  let tmDefinition = ''; 
  let savedFunctions = [];
  let functionCounter = 1;
  let addTransitionMode = false;
  let transitionSourceNode = null;

  let showTransitionModal = false;
  let transitionReadChar = '';
  let transitionWriteChar = '';
  let transitionMoveDirection = 'RIGHT';
  let transitionCallback = null;

  let showEdgeLabelModal = false;
  let edgeToEdit = null;
  let edgeLabel = '';

  let stateIdCounter = 0;

  let lastMousePosition = { x: 0, y: 0 };

  let contextMenuVisible = false;
  let contextMenuPosition = { x: 0, y: 0 };
  let contextNode = null;
  let contextEdge = null;
  let contextMenuType = '';

  let tape = [];
  let tapeHeadPosition = 0;
  let currentState = null;
  let isAcceptState = false;
  let isRunning = false;
  let simulationInterval = null;
  let transitionMap = {};

  cytoscape.use(cytoscapeCoseBilkent);
  cytoscape.use(undoRedo);

  function handleFileUpload(event) {
    const file = event.target.files[0];
    if (file) {
      const reader = new FileReader();
      reader.onload = (e) => {
        tmDefinition = e.target.result;
        parseAndRenderTMDefinition(tmDefinition);
      };
      reader.readAsText(file);
    }
  }

  function handleImageUpload(event) {
    const file = event.target.files[0];
    if (file) {
      const img = new Image();
      const reader = new FileReader();

      reader.onload = (e) => {
        img.src = e.target.result;
        img.onload = () => {
          processImage(img);
        };
      };
      reader.readAsDataURL(file);
    }
  }

  function processImage(img) {
    const nodes = cy.nodes();
    if (nodes.length === 0) {
      alert('No nodes available to rearrange. Please upload a Turing machine file first.');
      return;
    }

    const canvas = document.createElement('canvas');
    canvas.width = img.width;
    canvas.height = img.height;
    const ctx = canvas.getContext('2d');
    ctx.drawImage(img, 0, 0);
    const imageData = ctx.getImageData(0, 0, img.width, img.height);
    const data = imageData.data;

    const pixels = [];
    for (let y = 0; y < img.height; y++) {
      for (let x = 0; x < img.width; x++) {
        const index = (y * img.width + x) * 4;
        const r = data[index];
        const g = data[index + 1];
        const b = data[index + 2];
        const a = data[index + 3];

        if (a === 0) continue;

        const color = `rgba(${r}, ${g}, ${b}, ${a / 255})`;
        pixels.push({ x, y, color });
      }
    }

    if (pixels.length === 0) {
      alert('The image has no visible pixels to process.');
      return;
    }

    const gridSize=100;
    const cyContainer = cy.container();
    const containerWidth = cyContainer.clientWidth;
    const containerHeight = cyContainer.clientHeight;

    const imageAspectRatio = img.width / img.height;
    const containerAspectRatio = containerWidth / containerHeight;

    let scale;
    if (imageAspectRatio > containerAspectRatio) {
      scale = (containerWidth - gridSize * 2) / img.width;
    } else {
      scale = (containerHeight - gridSize * 2) / img.height;
    }

    const offsetX = (containerWidth - img.width * scale) / 2;
    const offsetY = (containerHeight - img.height * scale) / 2;

    const numNodes = nodes.length;
    const numPixels = pixels.length;

    const shuffledNodes = nodes.sort(() => Math.random() - 0.5);
    const shuffledPixels = pixels.sort(() => Math.random() - 0.5);

    const assignments = [];

    for (let i = 0; i < Math.min(numNodes, numPixels); i++) {
      assignments.push({ node: shuffledNodes[i], pixel: shuffledPixels[i] });
    }

    for (let i = Math.min(numNodes, numPixels); i < numNodes; i++) {
      assignments.push({
        node: shuffledNodes[i],
        pixel: null, 
      });
    }

    assignments.forEach(({ node, pixel }) => {
      if (pixel) {
        const x = Math.round((pixel.x * scale + offsetX) / gridSize) * gridSize;
        const y = Math.round((pixel.y * scale + offsetY) / gridSize) * gridSize;
        node.position({ x, y });
        node.data('color', pixel.color);
      } else {
        const x = Math.round(Math.random() * containerWidth / gridSize) * gridSize;
        const y = Math.round(Math.random() * containerHeight / gridSize) * gridSize;
        node.position({ x, y });
        node.data('color', '#fafad2'); 
      }
    });

    cy.batch(() => {
      nodes.emit('position');
    });
  }

  function parseAndRenderTMDefinition(tmDefinition) {
    const tmStates = [];
    const tmTransitions = [];
    let tapeLines = [];
    tapeHeadPosition = 0;

    const lines = tmDefinition.split('\n');
    let currentSection = null;

    for (let line of lines) {
      line = line.trim();
      if (line.startsWith('//') || line === '') {
        continue;
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
        const tokens = line.split(':');
        tapeHeadPosition = parseInt(tokens[1].trim());
        continue;
      }

      if (currentSection === 'STATES') {
        const tokens = line.split(/\s+/);
        const [id, x, y, start, accept, r, g, b, a] = tokens;
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
        const match = line.match(/^(\d+)\s(\d+)\s(.)\s(.)\s(\S+)$/);
    
    if (match) {
        const [_, fromStateId, toStateId, readChar, writeChar, moveDirection] = match;
        tmTransitions.push({
            from: fromStateId,
            to: toStateId,
            readChar: readChar,
            writeChar: writeChar,
            moveDirection: moveDirection,
        });
    } else {
        console.error(`Invalid transition line: ${line}`);
    }
      } else if (currentSection === 'TAPE') {
        if (!line.startsWith('Start Triangle Position:') && !line.startsWith('//') && line !== '') {
          if (isNaN(parseInt(line))) {
            tapeLines.push(line);
          }
        }
      }
    }

    const elements = [];

    tmStates.forEach((state) => {
      elements.push({
        data: {
          id: state.id,
          label: state.id,
          start: state.start,
          accept: state.accept,
          color: state.color,
        },
        position: { x: state.x, y: state.y },
      });
    });

    tmTransitions.forEach((transition) => {
      const transitionLabel = `${transition.readChar};${transition.writeChar};${transition.moveDirection}`;

      let edge = elements.find(
        (ele) =>
          ele.group === 'edges' &&
          ele.data.source === transition.from &&
          ele.data.target === transition.to
      );

      if (edge) {
        edge.data.label += `\n${transitionLabel}`;
      } else {
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

    initializeCytoscape(elements);

    tape = tapeLines.join('').split('');
    const startStates = tmStates.filter((state) => state.start);
    if (startStates.length > 0) {
      currentState = startStates[0].id;
    } else {
      currentState = tmStates[0].id;
    }
    isAcceptState = false;

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

    updateVisualization();
  }

  function initializeCytoscape(elements) {
    if (cy) {
      cy.destroy();
    }

    cy = cytoscape({
      container: document.getElementById('cy'),
      elements: elements,
      style: [
        {
          selector: 'node',
          style: {
            'background-color': 'data(color)',
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
            'overlay-opacity': 0.2,
            'overlay-color': '#dfeffc',
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
        name: 'preset',
      },
      selectionType: 'single',
      boxSelectionEnabled: true,
      autounselectify: false,
    });

    cy.userZoomingEnabled(true);
    cy.userPanningEnabled(true);

    ur = cy.undoRedo();

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
    ur.action(
  'changePosition',
  function (args) {
    const ele = args.ele;
    const newPosition = args.position;
    const oldPosition = { ...ele.position() };
    ele.position(newPosition);
    return { ele: ele, position: oldPosition };
  },
  function (args) {
    const ele = args.ele;
    const position = args.position;
    ele.position(position);
  }
);
    const numericIds = cy.nodes().map((node) => parseInt(node.id())).filter((id) => !isNaN(id));
    if (numericIds.length > 0) {
      stateIdCounter = Math.max(...numericIds) + 1;
    } else {
      stateIdCounter = 0;
    }

    cy.nodes().forEach((node) => {
      if (node.data('start')) {
        node.addClass('start');
      }
      if (node.data('accept')) {
        node.addClass('accept');
      }
    });

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

    document.addEventListener('keydown', handleKeyDown);

    cy.on('select', 'node', (event) => {
      console.log('Node selected:', event.target.id());
    });

    cy.on('free', 'node', (event) => {
      const node = event.target;
      const gridSize=100;
      const pos = node.position();
      const snappedPosition = {
        x: Math.round(pos.x / gridSize) * gridSize,
        y: Math.round(pos.y / gridSize) * gridSize,
      };
      node.position(snappedPosition);
    });
  }

  function addNodeAtPosition(x, y) {
    const gridSize=100;
    const newId = getNextAvailableStateId().toString();
    const snappedX = Math.round(x / gridSize) * gridSize;
    const snappedY = Math.round(y / gridSize) * gridSize;
    cy.add({
      group: 'nodes',
      data: { id: newId, label: newId, color: '#fafad2' },
      position: { x: snappedX, y: snappedY },
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

  function getNextAvailableStateId() {
    while (cy.getElementById(stateIdCounter.toString()).length > 0) {
      stateIdCounter++;
    }
    return stateIdCounter++;
  }

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

    const idMap = {};

    const elementsToAdd = [];

    selectedNodes.forEach((node) => {
      const data = { ...node.data() };
      const newId = getNextAvailableStateId().toString();
      idMap[node.id()] = newId;
      data.id = newId;
      data.label = newId;
      data.start = node.data('start');
      data.accept = node.data('accept');
      data.color = node.data('color');
      const gridSize=100; 
      const position = {
        x: Math.round((node.position('x') + 50) / gridSize) * gridSize,
        y: Math.round((node.position('y') + 50) / gridSize) * gridSize,
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

    ur.do('add', elementsToAdd);

    cy.elements().unselect();

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
      previewContainer: null,
    };
    savedFunctions = [...savedFunctions, func];
    console.log(`Saved new function: ${functionName}`);

    await tick();

    initializeFunctionPreview(func);
  }

  function initializeFunctionPreview(func) {
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
      autoungrabify: true, 
    });

    previewCy.nodes().forEach((node) => {
      if (node.data('start')) {
        node.addClass('start');
      }
      if (node.data('accept')) {
        node.addClass('accept');
      }
    });

    previewCy.on('tap', (event) => {
      event.preventDefault();
    });
  }

  function handleFunctionDragStart(func) {
    return (e) => {
      console.log(`Function drag started: ${func.name}`);
      e.dataTransfer.setData('functionIndex', savedFunctions.indexOf(func));
    };
  }

  function handleDrop(event) {
    event.preventDefault();
    const functionIndex = event.dataTransfer.getData('functionIndex');
    console.log(`Function dropped, index: ${functionIndex}`);
    if (functionIndex !== null && functionIndex !== '') {
      const func = savedFunctions[functionIndex];
      const rect = cy.container().getBoundingClientRect();

      const x = lastMousePosition.x - rect.left;
      const y = lastMousePosition.y - rect.top;

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

  function insertFunction(func, position) {
    console.log('Inserting function');

    const idMap = {};

    const funcNodes = func.elements.filter((ele) => ele.group === 'nodes');

    if (funcNodes.length === 0) return;

    const centroid = {
      x: funcNodes.reduce((sum, node) => sum + node.position.x, 0) / funcNodes.length,
      y: funcNodes.reduce((sum, node) => sum + node.position.y, 0) / funcNodes.length,
    };

    const offset = {
      x: position.x - centroid.x,
      y: position.y - centroid.y,
    };

    console.log(`Function centroid: (${centroid.x}, ${centroid.y}), offset: (${offset.x}, ${offset.y})`);

    const gridSize=100; 

    const elementsToAdd = func.elements.map((ele) => {
      const data = { ...ele.data };
      if (ele.group === 'nodes') {
        const newId = getNextAvailableStateId().toString();
        idMap[data.id] = newId;
        data.id = newId;
        data.label = newId;
        data.start = ele.data.start;
        data.accept = ele.data.accept;
        data.color = ele.data.color;
        ele.data = data;
        ele.position = {
          x: Math.round((ele.position.x + offset.x) / gridSize) * gridSize,
          y: Math.round((ele.position.y + offset.y) / gridSize) * gridSize,
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

    ur.do('add', elementsToAdd);
  }

  function parseColorString(colorStr) {
    let r = 0.98,
      g = 0.98,
      b = 0.82,
      a = 1;
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

  function exportTMDefinition() {
    let output = '';
    output += '// Save File for STEM\n';
    output += '// Version 1.00\n\n';

    output += '// State Format: name x y start accept\n';
    output += 'STATES:\n';

    const containerHeight = cy.container().offsetHeight;

    cy.nodes().forEach((node) => {
      const id = node.id();
      const x = node.position('x');
      const y = node.position('y');

      const invertedY = Math.abs(y);

      const start = node.data('start') ? 'true' : 'false';
      const accept = node.data('accept') ? 'true' : 'false';
      const colorStr = node.data('color') || '#fafad2';
      const rgba = parseColorString(colorStr);
      output += `\t${id} ${x} ${invertedY} ${start} ${accept} ${rgba.r} ${rgba.g} ${rgba.b} ${rgba.a}\n`;
    });


    output += '\n// Transition format: fromStateId toStateId readChar writeChar moveDirection\n';
    output += "// The Character '~' is the catchall character\n";
    output += 'TRANSITION:\n';

    cy.edges().forEach((edge) => {
    const source = edge.source().id();
    const target = edge.target().id();
    const labels = edge.data('label').split('\n');

    labels.forEach((label) => {
        const [readChar, writeChar, moveDirection] = label.split(';').map((s) => s === ' ' ? ' ' : s.trim());
        
        const formattedReadChar = readChar === ' ' ? " " : readChar;
        const formattedWriteChar = writeChar === ' ' ? " " : writeChar;

        output += `\t${source} ${target} ${formattedReadChar} ${formattedWriteChar} ${moveDirection}\n`;
    });
});

    output += '\n// Tape format: tapeChar(0) tapeChar(1) ... tapeChar(n)\n';
    output += 'TAPE:\n';
    output += `\t${tapeHeadPosition}\n`;
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
  function snapToGrid() {
  const gridSize=100; 
  const snapGridSize = gridSize / 5;
  let nodesToSnap = cy.nodes(':selected');
  if (nodesToSnap.length === 0 && contextNode) {
    nodesToSnap = cy.collection([contextNode]);
  }
  if (nodesToSnap.length === 0) {
    console.log('No nodes selected to snap to grid');
    return;
  }
  ur.do('batch', () => {
    nodesToSnap.forEach((node) => {
      const newPosition = {
        x: Math.round(node.position('x') / snapGridSize) * snapGridSize,
        y: Math.round(node.position('y') / snapGridSize) * snapGridSize,
      };
      ur.do('changePosition', { ele: node, position: newPosition });
    });
  });
  closeContextMenu();
}
  function addOrUpdateEdge(sourceId, targetId, readChar, writeChar, moveDirection) {
    let edge = cy
      .edges()
      .filter((edge) => edge.source().id() === sourceId && edge.target().id() === targetId)
      .first();

    const transitionLabel = `${readChar};${writeChar};${moveDirection}`;

    if (edge && edge.length > 0) {
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

  onMount(() => {
    initializeCytoscape([]);

    const cyContainer = document.getElementById('cy');
    cyContainer.addEventListener('dragover', handleDragOver);
    cyContainer.addEventListener('drop', handleDrop);
    console.log('Cytoscape initialized and event listeners added');
  });

  function updateVisualization() {
  }

</script>

<div>
  <input type="file" on:change={handleFileUpload} accept=".txt" />
  <button on:click={exportTMDefinition}>Export Turing Machine</button>
  <button on:click={startAddTransition}>Add Transition</button>
</div>

<div>
  <input type="file" on:change={handleImageUpload} accept="image/*" />
  <label>Upload Image to Rearrange Nodes</label>
</div>

<div id="cy"></div>

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
        <li on:click={snapToGrid}>Snap to Grid</li>

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
  #cy {
    width: 100%;
    height: 70vh;
    border: 1px solid #ccc;
    /* background-image:
      linear-gradient(to right, #eee 1px, transparent 1px),
      linear-gradient(to bottom, #eee 1px, transparent 1px); */
    background-size: 20px 20px;
  }

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
    pointer-events: none;
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
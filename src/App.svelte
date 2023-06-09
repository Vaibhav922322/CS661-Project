<script>
	import { givenInstanceIdGetLeafNodeMap as IdToLeafNodeMap } from "./util";
	import { ScaleOut } from "svelte-loading-spinners";
	import { imagesEndpoint } from "./stores/endPoints";
	import {
		globalClasses,
		globalLeafNodesObject,
	} from "./stores/globalDataStore";
	import {
		treemapNumClusters,
		treemapImageSize,
		hasClasses,
		hasTrueClass,
		hasPredictedClass,
		hasSimilar,
		hasAccuracy,
		selectedImage,
		imagesToHighlight,
		highlightSimilarImages,
		highlightIncorrectImages,
		showMisclassifications,
		selectedParent,
	} from "./stores/sidebarStore";
	import * as links from "./links";

	import Sidebar from "./components/Sidebar.svelte";
	import DendroMap from "./components/dendroMap/DendroMap.svelte";
	import GithubIcon from "./components/misc/GithubIcon.svelte";
	import PaperIcon from "./components/misc/PaperIcon.svelte";
	import ArticleSidebar from "./components/article/ArticleSidebar.svelte";

	// check (stores/globalDataStore.js for more info.)
	function storeDataGlobally({ classes, leafNodes, leafIdMap }) {
		globalLeafNodesObject.set({ idMap: leafIdMap, array: leafNodes });
		globalClasses.set(classes);
	}

	function processData(tree) {
		function getLeafNodes(node) {
			const leaves = [];
			function _getLeafNode(parent) {
				if (parent.leaf || parent.children === undefined) {
					leaves.push(parent);
					return;
				}
				parent.children.forEach((child) => _getLeafNode(child));
			}
			_getLeafNode(tree);
			return leaves;
		}
		const leafNodes = getLeafNodes(tree);
		const leafIdMap = IdToLeafNodeMap(leafNodes);

		return { leafIdMap, leafNodes };
	}
	async function formatAndStoreDendrogram(tree, classes) {
		rootNode = tree;
		const { leafIdMap, leafNodes } = processData(tree);
		// change the visualization based on provided information
		const firstLeafNode = leafNodes[0];
		hasSimilar.set("similar" in firstLeafNode);
		hasPredictedClass.set("predicted_class" in firstLeafNode);
		hasTrueClass.set("true_class" in firstLeafNode);
		hasAccuracy.set("accuracy" in firstLeafNode);
		let hasClassesValue = classes !== undefined;
		hasClasses.set(hasClassesValue);
		if (hasClassesValue) {
			treeClasses = classes;
		}
		let output = {
			classes: treeClasses,
			leafNodes,
			leafIdMap,
		};

		storeDataGlobally(output);
		imagesEndpoint.set(selectedOption.image_filepath);
	}
	async function fetchData() {
		showTreemap = await false;
		if (dataCache === null) {
			const res = await fetch(selectedOption.cluster_filepath);
			const data = await res.json();
			dataCache = data;
		}
		console.log(selectedOption.image_filepath);
		await formatAndStoreDendrogram(
			dataCache.tree,
			dataCache.classes ?? undefined
		);
		showTreemap = await true;
	}
	async function fetchClassedData(selectedClass) {
		showTreemap = await false;
		classClusteringsPresent = false;
		if (!(selectedClass in classedDataCache)) {
			const res = await fetch(selectedOption.class_cluster_filepath);
			const data = await res.json();
			classedDataCache = {};
			data["classes"].forEach((class_name) => {
				const tree = data[class_name];
				classedDataCache[class_name] = {
					tree,
					classes: data["classes"],
				};
			});
		}
		const selectedData = classedDataCache[selectedClass];
		formatAndStoreDendrogram(selectedData.tree, selectedData.classes);
		classClusteringsPresent = true;
		showTreemap = await true;
	}
	function silenceConsoleLogs() {
		console.log("console log is silenced 😴");
		console.log = () => {};
	}

	/**
	 * Takes in an object with keys that are the URL param name and the value is a callback that contains the url value
	 * @param {paramName: (value) => void} requestedParamsObj
	 */
	function getURLParameters(requestedParamsObj) {
		const requestedEntries = Object.entries(requestedParamsObj);
		const urlParameters = new URLSearchParams(window.location.search);
		requestedEntries.forEach(([parameter, callback], i) => {
			if (urlParameters.has(parameter)) {
				const value = urlParameters.get(parameter);
				callback(value);
			} else {
				callback(undefined);
			}
		});
	}

	// props
	export let options; // settings you can change in main.js that shows up in the dropdown in the sidebar

	getURLParameters({
		mnist: (value) => {
			if (value !== undefined) {
			} else {
				options = options.filter(
					(option) => option.dataset.toLowerCase() !== "mnist"
				);
			}
		},
	});
	export let silenceConsole = false;
	if (silenceConsole) {
		silenceConsoleLogs();
	}

	// vars
	let selectedOptionIndex = 0;
	let selectedOption;
	$: {
		selectedOption = options[selectedOptionIndex];
		classedDataCache = {};
		dataCache = null;
	}

	let classedDataCache = {};
	let dataCache = null;
	let currentParentCluster = null;

	// indicators of when things are done or if we have a certain item
	let changedDataset = false;
	let articleOpen = false;
	let showTreemap = false;
	let classClusteringsPresent;

	// dendromap dimension size
	const screen = {
		width: document.body.clientWidth,
		height: document.body.clientHeight,
	};

	// app variables for data
	let dendrogramData;
	let treeClasses;
	let rootNode;

	// on change of the dataset update the dataset
	$: {
		const updateSelection = async (index) => {
			changedDataset = await true;
			await fetchData(index);
			changedDataset = await false;
		};
		updateSelection(selectedOptionIndex);
	}
	$: {
		selectedParent.set(currentParentCluster);
	}
</script>

<div>
	<div id="main">
		<div id="sidebar">
			<Sidebar
				on:filterClass={async (e) => {
					const className = e.detail;
					showTreemap = await false;
					if (className === null) {
						await fetchData();
					} else {
						await fetchClassedData(className);
					}
					console.log(e.detail);
					showTreemap = await true;
				}}
				classes={treeClasses}
				{options}
				bind:selectedOption={selectedOptionIndex}
				bind:articleSidebarOpen={articleOpen}
				{changedDataset}
			/>
		</div>
		<div id="vis">
			{#if showTreemap}
				<DendroMap
					dendrogramData={rootNode}
					imageFilepath={selectedOption.image_filepath}
					imageWidth={$treemapImageSize}
					imageHeight={$treemapImageSize}
					width={Math.max(screen.width - 600, 600)}
					height={835}
					renderingMethod={"breadth"}
					numClustersShowing={$treemapNumClusters}
					imagesToFocus={$imagesToHighlight}
					outlineMisclassified={$showMisclassifications}
					focusMisclassified={$highlightIncorrectImages}
					clusterLabelCallback={(d) => {
						let totalLabel = `${d.data.node_count} image${
							d.data.node_count > 1 ? "s" : ""
						}`;
						if ($hasAccuracy) {
							totalLabel += `, ${(d.data.accuracy * 100).toFixed(
								2
							)}% accuracy`;
						}
						return totalLabel;
					}}
					imageTitleCallback={(d) => {
						let titleMsg = `Click to select image ${d.instance_index}`;
						if ($hasTrueClass) {
							titleMsg += `\ntrue class: ${d.true_class}`;
						}
						if ($hasPredictedClass) {
							titleMsg += `\npred class: ${d.predicted_class}`;
						}
						return titleMsg;
					}}
					bind:currentParentCluster
					on:imageClick={(e) => {
						const { data, el, event } = e.detail;
						selectedImage.set(data); // pass to the sidebar
					}}
					on:imageMouseEnter={(e) => {
						const { data, el, event } = e.detail;
						if ($highlightSimilarImages) {
							imagesToHighlight.set([
								data.instance_index,
								...data.similar,
							]);
						}
					}}
					on:imageMouseLeave={(e) => {
						const { data, el, event } = e.detail;
						if ($highlightSimilarImages) {
							imagesToHighlight.set([]);
						}
					}}
					on:clusterClick={(e) => {
						// const { data, el, event } = e.detail;
					}}
					on:clusterMouseEnter={({ detail }) => {
						// const { data, el, event } = e.detail;
					}}
					on:clusterMouseLeave={({ detail }) => {
						// const { data, el, event } = e.detail;
					}}
				/>
			{:else}
				<div style="display:flex; gap:10px; align-items:center;">
					<h1 style="margin:0;padding:0;color: #00000020;">
						Loading Data
					</h1>
					<ScaleOut size="40" color="#333333" unit="px" />
				</div>
			{/if}
		</div>
	</div>
	<!--<ArticleSidebar bind:open={articleOpen} />-->
</div>

<style>
	#top-bar {
		width: 100%;
		height: 25px;
		background-color: var(--dark-grey);
		padding-top: 10px;
		padding-bottom: 10px;
		display: flex;
		align-items: center;
	}
	#title {
		color: white;
		font-size: 25px;
		font-weight: 600;
		margin-left: 20px;
		margin-top: -4px;
	}
	#main {
		display: flex;
		height:835px;
		
		border-bottom: 1.5px solid #00000010;

		 background: linear-gradient(-45deg, #ee7752, #e73c7e, #23a6d5, #23d5ab);
	background-size: 400% 400%;
	animation: gradient 15s ease infinite;

	
}

@keyframes gradient {
	0% {
	 background-position: 0% 50%;
	}
		
	50% {
		background-position: 100% 50%;
	}
		
	100% {
		background-position: 0% 50%;
	}
}


	#sidebar {
		--width: 550px;
		width: var(--width);
		max-width: var(--width);
		min-width: var(--width);
	}
	code {
		font-family: monospace;
	}
	#links {
		display: flex;
		color: white;
		position: absolute;
		right: 25px;
	}
	#vis {
		width: 100%;
		display: flex;
		justify-content: center;
		align-items: center;
	}
</style>

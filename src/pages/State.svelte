<script>
// library, model, and local app store
import { push } from 'svelte-spa-router';
import { onMount, onDestroy } from 'svelte';

import moment from 'moment';
import model from '../model.js';
import { currentPage, pageTitle, currentState } from '../stores.js';

// presentational components
import Tab, {Icon, Label} from '@smui/tab';
import TabBar from '@smui/tab-bar';
import Dialog, {Content, Actions, Title} from '@smui/dialog';
import LinearProgress from '@smui/linear-progress';

import BarLineChart from '../components/BarLineChart.svelte';
import DisplayTable from '../components/DisplayTable.svelte';

export let params;
export let state;


let types = {
    'Confirmed Cases (daily)': { type: 'positiveIncrease', barColor: '#a3bef9', lineColor: '#f57d00' },
    'Mortalities (daily)': { type: 'deathIncrease', barColor: '#f79494', lineColor: '#02b147' },
    'Tested (daily)': { type: 'totalTestResultsIncrease', barColor: '#99d493', lineColor: '#f57d00' },
    'Hospitalized': { type: 'hospitalizedCurrently', barColor: '#ffb97c', lineColor: '#7cc2ff'},
    'In ICU': { type: 'inIcuCurrently', barColor: '#ffb97c', lineColor: '#7cc2ff'},
    'On Ventilator': { type: 'onVentilatorCurrently', barColor: '#ffb97c', lineColor: '#7cc2ff'},
    'Confirmed Cases': { type: 'positive', barColor: '#a3bef9', lineColor: '#f57d00' },
    'Mortalities': { type: 'death', barColor: '#f79494', lineColor: '#02b147'},
    'Tested': { type: 'totalTestResults', barColor: '#99d493', lineColor: '#f57d00' }
};
let activeTab = Object.keys(types)[0];

let curType = types[activeTab];

let stateName = '';
let states = Object.entries( model.getStates() || {})
    // Exclude entities with no data
    .filter( state => !['FM', 'MH', 'PW'].includes(state[0]) )
    .reduce( (acc, state) => {
        acc[state[0]] = state[1];
        return acc;
    }, {});


let covidData = { headers: [], data: [] };
let tableData = {};
let chartData = [];

// For table sorting
let activeColumn = 'date';
let curDir = 'desc';

// local dialog variables
let dialogRef;
let dialogTitle = '';
let dialogContent = '';

let isFetchingData = false;

// Setting the state triggers changes on the page, including changes to covidData
$: {
    (async () => {
        if(dialogRef == undefined) { return; }
        isFetchingData = true;

        state = params.state;
        stateName = states[state];
        
        pageTitle.set(stateName);
        currentState.set(state);

        dialogTitle = "Please wait";
        dialogContent = "Loading";
        
        // There's some sort of focus bug when opening up the dialog sometimes, what could be going on here?
        try { dialogRef.open(); }
        catch(e) { console.error(e); }

        try {
            covidData = await getData(state);
            dialogRef.close();
        }
        catch(e) {
            dialogTitle = "Error";
            dialogContent = "There was an error loading the page, please try reloading your browser";
            console.error('[State] There was an error fetching the data', e);
        }
        isFetchingData = false;
    })();
};

// Set the active column and direction from the params
$: { 
    if(params) {
        let { sort, dir } = params;
        activeColumn = sort;
        curDir = dir;
    }
};

// Whenever activeTab changes, update the chartData
$: curType = types[activeTab];

// Set up tableData whenever covidData changes
$: { 
    let newData = covidData.data.slice();
    newData = newData.sort(sortData(activeColumn, curDir) );
    tableData = getTableData(covidData.headers, newData);
}

// Set up chart
$: chartData = getChartData([...covidData.data], curType);


const getChartData = (data, activeType) => {
    if(!Array.isArray(data) ) { return []; }

    let attributes, 
        barColor,
        lineColor,
        xData = [], barData = [], lineData = [],
        yMinMax = { min: 0, max: 0 };
    

    let yReducer = list => (acc, cur) => {
        for(let i=0; i < list.length; i++) {
            // if(!acc) { acc = []; }
            acc.push(cur[list[i]]);
        }
        return acc;
    };

    let getRollingAvgs = list => {
        let rollingSum = 0;
        let rollingAvgs = [];
        for(let i=0; i < list.length; i++) {
            rollingSum += list[i];
            if(i >= 7) { 
                rollingSum -= list[i-7]; 
                rollingAvgs[i] = Math.round(rollingSum / 7);
            }
            else {
        //        rollingAvgs[i] = rollingSum / (i+1);
                rollingAvgs[i] = 0;
            }
        }
        return rollingAvgs;
    };


    attributes = (activeType.type === 'positive_and_death') ? 
        ['positive', 'death'] : [activeType.type];
    barColor = activeType.barColor;
    lineColor = activeType.lineColor;

    xData = data.map(row => new Date(row.date) );
    barData = data.reduce( yReducer(attributes), []);
    lineData = getRollingAvgs(barData);

    yMinMax = { min: Math.min(...barData.flat() ), max: Math.max(...barData.flat() ) };

    return { 
        xData, 
        barData,
        lineData,
        labels: { 'x': 'date', 'y': attributes },
        barColor,
        lineColor,
        min: { x: xData[0], y: yMinMax.min},
        max: { x: xData[xData.length - 1], y: yMinMax.max}
    };    
};

const getTableData = (theHeaders, theData) => {
    if(theHeaders == undefined || theData == undefined) 
    { console.error(`[State.getTableData] No headers/data provided`); return; }

    let newData = { 
        headers: theHeaders.slice(),
        data: theData.slice()
    };

    // For creating the cells in each row of the table
    let tableCells = newData.headers.map(entry => entry.data);
    
    newData.data = theData.map(entry => {
        let newRow = {};
        for(let prop of tableCells) {
            newRow[prop] = (isNaN(entry[prop])) ? entry[prop] : formatNumber(entry[prop]); 
        }

        let { date, day, positiveIncrease, deathIncrease, totalTestResultsIncrease } = entry;

        // custom attributes
        newRow['date'] = `${date} (${day})`;
        newRow['positive'] += ` (${formatNumber(positiveIncrease)})`;
        newRow['death'] += ` (${formatNumber(deathIncrease)})`; 
        newRow['totalTestResults'] += ` (${formatNumber(totalTestResultsIncrease)})`; 

        return newRow;
    });
    return newData;
};

// Helper function for formatting numbers
const formatNumber = number => new Intl.NumberFormat().format(number);

const sortData = (column, direction) => (first, second) => {
    let multiplier = (direction === 'asc') ? 1 : -1;
    return (first[column] < second[column]) ? multiplier * -1 : multiplier * 1;
};

const getData = async curState => {
    if(!curState) { console.error('[State.getData] No state present'); return; }
    return await model.get({ type: 'state', state: curState });
}

// Handles the sorting that occurs when a user clicks on the Header
const dispatchHandlerHeaderClick = e => {

    let target = e.detail.target;
    if(!target) { console.error('[State.dispatchHandlerHeaderClick] Unable to sort', event.target); return; }

    let index = target.dataset.index;
    let newColumn = target.dataset.sort;

    // If switching columns, use the new columns default sort
    // Else, same column, toggle between asc/desc
    curDir = (activeColumn !== newColumn) ? covidData.headers[index].defaultSort : 
        ((curDir === 'asc') ? 'desc' : 'asc');
    activeColumn = newColumn;
    
    push(`/state/${state}/sort/${activeColumn}/${curDir}`);    
};

const dispatchHandlerRowClick = e => {
    let target = e.detail.target;
    if(!target) { console.error('[State.dispatchHandlerRowClick] Unable to sort', event.target); return; }
};

const handlerKeydown = function(e) {
    if(isFetchingData === true) { return; }
    if(!['ArrowUp', 'ArrowDown'].includes(e.key)) { return; }

    let modifier = e.key === 'ArrowDown' ? +1 : -1;
    let stateList = Object.keys(states);
    let index = stateList.indexOf($currentState);

    if(index >= 0) {
        // increment the state list by +1/-1 depending on ArrowUp/ArrowDown
        index += modifier;
        
        // keep the index inbounds, cycle around to the other side
        index = (index > stateList.length - 1) ? 0 : index;
        index = (index < 0) ? stateList.length - 1 : index;

        if(activeColumn != 'undefined' && ['asc','desc'].includes(curDir)) {
            push(`/state/${stateList[index]}/sort/${activeColumn}/${curDir}`);    
        }
        else {
            push(`/state/${stateList[index]}/`);    
        }
    }
}

const handlerDialogClosed = function(e) { };

onMount( () => {
    state = params.state;
    stateName = states[state];        

    // set the app store states
    pageTitle.set(stateName);
    currentState.set(state);
    currentPage.set('state');
});

</script>
<style type="text/scss">
    .content {
        display: flex;
        flex-direction: column; 
    }
    // Style for the child table
    .content :global(.mdc-data-table th:hover) {
        cursor: pointer;
    }

    .charts { 
        display: flex;
        flex-direction: column;
        max-height: 40vh;
    }

	@media (min-width: 640px) {
        .content {
            flex-direction: row;
        }
        .charts { 
            max-width: 50%;
            max-height: initial;
        }
    }
</style>

<svelte:head>
    <title>CV Totals for {stateName}</title>
</svelte:head>

<svelte:body on:keydown|preventDefault={handlerKeydown} />

{#if Object.keys(covidData).length > 0}
<section class="content">
    <section class="charts">
        <TabBar tabs={Object.keys(types)} let:tab bind:active={activeTab} >
            <!-- Notice that the `tab` property is required! -->
            <Tab {tab}>
                <Label>{tab}</Label>
            </Tab>
        </TabBar>
        <BarLineChart dataset={chartData} />
        <!-- <LineChart dataset={chartData} /> -->
    </section>
    <DisplayTable dataset={tableData} 
        on:headerClick={dispatchHandlerHeaderClick} 
        on:rowClick={dispatchHandlerRowClick} 
    />
</section>
{/if}

<Dialog bind:this={dialogRef} 
    aria-labelledby="simple-title" 
    aria-describedby="simple-content"
    on:MDCDialog:closed={handlerDialogClosed}
>
    <!-- Title cannot contain leading whitespace due to mdc-typography-baseline-top() -->
    <Title id="simple-title">{dialogTitle}</Title>
    <Content id="simple-content">{dialogContent}</Content>
    <LinearProgress indeterminate />
</Dialog>

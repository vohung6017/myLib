** make sure one function only process 1 feature to easier maintain.
** short and simple fuction name
** ES6+
** comment form (no need //==============):
// comment
*global variable
*helper
*utils
*fetch data from api
*process data from fetching
*load,render data to tables/charts...
*onclick functions (each function use for 1 part of code. eg: click event of modal,table... )

const loadData = () => {
    load and render data here
}

const loadEvent = () => {
    onclick functions here
}

document.addEventListener("DOMContentLoaded", () => {
    loadData();
    loadEvent();
});

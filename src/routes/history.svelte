<script>
  /**
   * History Tab
   * A big collection of all things history
   *
   * TODO: Have it react when the ledger change, not a hard refresh
   */

  // svelte
  import { navigate } from "svelte-routing";
  import { onMount } from "svelte";
  import { fly } from "svelte/transition";

  // components
  import NItem from "../components/list-item/list-item.svelte";
  import NCell from "../components/cell/cell.svelte";
  import NText from "../components/text/text.svelte";
  import NPoints from "../components/points/points.svelte";
  import NNoteTextualizer from "../components/note-textualizer/note-textualizer.svelte";
  import NToolbar from "../components/toolbar/toolbar.svelte";
  import NModal from "../components/modal/modal.svelte";
  import Spinner from "../components/spinner/spinner.svelte";
  import NDatePicker from "../components/date-picker/date-picker.svelte";
  import LogItem from "../components/list-item-log/list-item-log.svelte";
  import NDatePill from "../components/date-pill/date-pill.svelte";

  // Containers
  import NMap from "../containers/map/map.svelte";
  import AppLayout from "../containers/layout/app.svelte";

  // Utils
  import NomieUOM from "../utils/nomie-uom/nomie-uom";
  import dayjs from "dayjs";

  // Stores
  import { UserStore } from "../store/user";
  import { Interact } from "../store/interact";
  import { TrackerStore } from "../store/trackers";
  import { LedgerStore } from "../store/ledger";
  import { Lang } from "../store/lang";

  import { HistoryPage } from "../store/history-page";

  /**
   * I've messed this all up again. but it's faster and more responsivle
   * TODO: refactor so it's clean and using the proper amount of Store vs local
   */

  let datePicker;
  let searchInput;
  let appTitle = null;

  const state = {
    date: dayjs(new Date()),
    time_format: "YYYY-MM",
    logs: null,
    trackers: {},
    ledger: null,
    searchTerm: "",
    searchResults: null,
    searchMode: false,
    selected: {},
    selectCount: 0,
    editMode: false,
    showDatePicker: false,
    location: {
      name: null,
      lat: null,
      lng: null
    },
    locations: [],
    loading: true,
    showAllLocations: false,
    searchYear: dayjs().format("YYYY")
  }; // Assign State to compiled history page

  let refreshing = false;

  let local = {
    showDatePicker: false,
    datePickerValue: null,
    searchMode: false
  };

  // $: searchMode = (state.searchTerm || "").length ? true : false;
  let searchMode = false;
  $: if (state.searchTerm && !searchMode) {
    searchMode = true;
  }

  let logs = undefined; // holder of the logs
  let searchLogs = undefined; // hodler of searched logs
  let loading = true;
  let book = undefined;
  let locations = [];
  let dayScore = 0;

  // Used for checking things
  const checks = {
    list_date: {}
  };

  /// Watchers for when we're in edit mode
  // and when we have selected more than one.

  let isToday = true;

  // If the date changes - check to see if it's still today
  let activeDate;
  $: if (state.date && state.date !== activeDate) {
    activeDate = state.date;
    isToday = new Date().toDateString() == state.date.toDate().toDateString();
  }

  // Filter logs for today
  const filterActiveDate = log => {
    return (
      log.end >=
        state.date
          .startOf("day")
          .toDate()
          .getTime() &&
      log.end <=
        state.date
          .endOf("day")
          .toDate()
          .getTime()
    );
  };

  // Dynamically assign book
  $: if ($LedgerStore.books[state.date.format("YYYY-MM")] && !searchMode) {
    loading = true;
    book = $LedgerStore.books[state.date.format("YYYY-MM")] || [];
    logs = (book || [])
      .filter(log => filterActiveDate(log))
      .sort((a, b) => {
        return a.end < b.end ? 1 : -1;
      });

    dayScore = 0;
    logs
      .filter(log => {
        return log.score;
      })
      .forEach(log => {
        dayScore = dayScore + parseInt(log.score);
      });

    loading = false;
  }

  $: appTitle = `History ${state.date.format("YYYY-MM-DD")}`;

  $: if (searchLogs || logs) {
    locations = (searchLogs || logs)
      .filter(log => {
        return log.lat;
      })
      .map(log => {
        return {
          lat: log.lat,
          lng: log.lng,
          name: log.location
        };
      });
  }

  // Methods
  const methods = {
    toggleSearch() {
      if (searchMode) {
        state.searchResults = null;
        state.searchTerm = null;
        searchMode = false;
      } else {
        searchMode = true;
      }
    },
    getLogs(fresh) {
      fresh = fresh ? fresh : false;

      loading = true;
      // state.refreshing = true;
      // checks.list_date = {};

      // Query the Ledger for Posts on this day.
      return LedgerStore.query({
        start: state.date.startOf("day").toDate(),
        end: state.date.endOf("day").toDate(),
        fresh: fresh
      });
    },
    clearLocation() {
      state.location.name = null;
      state.location.lat = null;
      state.location.lng = null;
    },
    refresh() {
      refreshing = true;
      setTimeout(() => {
        refreshing = false;
      });
    },
    clearSearch() {
      state.searchResults = null;
      searchMode = false;
      state.searchTerm = "";
      searchLogs = null;
    },
    previous() {
      methods.getDate(state.date.subtract(1, "day"));
    },
    getDate(date) {
      state.date = date;
      methods.getLogs();
    },
    next() {
      methods.getDate(state.date.add(1, "day"));
    },
    previousSearch() {
      state.date = state.date.subtract(1, "year").startOf("year");
      methods.search(state.searchTerm, state.date.format("YYYY"));
    },
    nextSearch() {
      state.date = state.date.add(1, "year").startOf("year");
      methods.search(state.searchTerm, state.date.format("YYYY"));
    },
    headerExists(date) {
      // let dformat = dayjs(date).format("YYYY-MM-DD");
      // if (checks.list_date.hasOwnProperty(dformat)) {
      //   return true;
      // } else {
      //   checks.list_date[dformat] = true;
      //   return false;
      // }
    },
    goto(date) {
      state.date = date;
      methods.getLogs();
    },
    formSubmit(event) {
      event.preventDefault();
      event.stopPropagation();
      methods.search(state.searchTerm, state.date.format("YYYY"));
      searchMode = true;
    },
    searchKeypress(event) {
      if (event.key === "Enter" || event.key === "Return") {
        methods.search(state.searchTerm, state.date.format("YYYY"));
        return false;
      }
    },
    refreshSearch() {
      methods.search(state.searchTerm, state.date.format("YYYY"));
    },
    search(key, year) {
      searchMode = true;
      if ((key || "").length > 1) {
        state.searchResults = [];
        loading = true;
        LedgerStore.search(state.searchTerm, year).then(searchResults => {
          loading = false;
          searchLogs = searchResults;
        });
      }
    },
    trackerTapped(tracker, log) {
      // let trackerId = ($TrackerStore[tracker.tag] || {}).id;
      // navigate(`/stats/${tracker.tag}`);
      Interact.openStats(tracker.tag);
    },
    showLogOptions(log) {
      Interact.logOptions(log).then(action => {
        if (searchMode) {
          methods.refreshSearch();
        }
      });
    },
    selectDate() {
      let ranges = [
        {
          days: 90,
          title: "90 Days Back"
        },
        {
          days: 180,
          title: "6 Months Back"
        },
        {
          days: 365,
          title: "1 Year Back"
        },
        {
          days: 730,
          title: "2 Years Back"
        },
        {
          days: -1,
          title: "Select Date..."
        }
      ];

      if (!isToday) {
        ranges.unshift({
          days: 0,
          title: "Go to Today"
        });
      }

      Interact.popmenu({
        buttons: ranges.map(range => {
          return {
            title: range.title,
            click() {
              if (range.days == -1) {
                local.showDatePicker = true;
              } else if (range.days === 0) {
                methods.goto(dayjs());
              } else {
                methods.goto(state.date.subtract(range.days, "days"));
              }
            }
          };
        })
      });
    }
  };

  onMount(() => {
    if ((state.searchTerm || "").length > 1 && !searchLogs) {
      methods.refreshSearch();
    }
    window.scrollTo(0, 0);
  });

  /**
   * // Fire off call to query the datastore
   * This will call the ledger and load up the right book
   * once the book is loaded, the logs var will be automaticallly filtered
   */
  methods.getLogs();
</script>

<style lang="scss" type="text/scss">
  // TODO: This is really sloppy - clean it up.
  hr {
    margin: 0;
    padding: 0;
    border-top: solid 1px rgba(0, 0, 0, 0.2);
  }
  :global(.trackers-list .border-bottom:last-child) {
    border-bottom: none !important;
  }

  .history-toolbar-container {
    .btn {
      outline: none !important;
    }
    .btn.active {
      color: var(--color-primary-bright) !important;
      padding-top: 5px;
      padding-bottom: 5px;
      border-radius: 20% !important;
      outline: none !important;
    }
  }

  .n-date-pill-container {
    position: fixed;
    bottom: 70px;
    left: 0;
    right: 0;
  }

  .header-date-control {
    line-height: 100%;
    flex-grow: 1;
    flex-shrin: 1;
  }

  :global(.trackers-list) {
    border-bottom: solid 1px rgba(0, 0, 0, 0.06);
    border-top: solid 1px rgba(0, 0, 0, 0.06);
    margin-top: 10px;
  }
  :global(.trackers-list .n-item) {
    margin-left: -16px;
    margin-right: -16px;
    padding-left: 16px !important;
    padding-right: 16px !important;
    border-bottom-color: rgba(0, 0, 0, 0.05) !important;
  }
  :global(.trackers-list .main div) {
    font-size: 1.1rem !important;
  }
  :global(.trackers-list .emoji) {
    font-size: 1.2rem !important;
  }
  .btn.flex {
    display: flex;
    align-items: center;
  }

  :global(.page-history .n-item .n-item:last-child) {
    border-bottom: none !important;
  }
</style>

<AppLayout title={appTitle}>

  <header slot="header">
    <NToolbar>
      <div class="container history-toolbar-container">
        <div class="d-flex justify-content-stretch align-items-center w-100">
          {#if searchMode}
            <button
              class="btn btn-clear btn-icon flex"
              on:click={methods.previousSearch}>
              <i class="zmdi zmdi-arrow-left" />
            </button>
          {:else}
            <button
              class="btn btn-clear btn-icon flex"
              on:click={methods.previous}>
              <i class="zmdi zmdi-arrow-left" />
            </button>
          {/if}
          <!-- <div class="filler" /> -->
          <!-- <button
        class="btn btn-clear btn-icon flex {searchMode ? 'active text-primary-bright' : ''}"
        on:click={methods.toggleSearch}>
        <i class="zmdi zmdi-search" />
      </button>
      <div class="filler" /> -->
          {#if searchMode}
            <div class="filler" />
            <div class="text-center n-text md">
              Search {state.date.format('YYYY')}
            </div>
            <div class="filler" />
          {:else}
            <div class="filler" />

            <div
              class="header-date-control justify-content-center flex-grow
              text-center"
              on:click={methods.selectDate}>
              <div class="text-center text-md n-row">
                {#if dayScore}
                  <NPoints points={dayScore} className="mr-2" />
                {:else}
                  <i class="zmdi mr-4 text-faded-3 text-xs" />
                {/if}
                <div class="col">
                  <div
                    class="{isToday ? 'text-inverse' : 'not-today text-primary-bright'}
                    text-md text-inverse font-weight-bold md">
                    {state.date.format('dddd')}
                  </div>
                  <div class="text-sm text-inverse-2">
                    {state.date.format('MMM D YYYY')}
                  </div>
                </div>
                <i class="zmdi zmdi-calendar mx-2 text-faded-3 text-xs" />
              </div>
              <!-- end text middle -->
            </div>
            <!-- end header-date-control -->

            <div class="filler" />
          {/if}
          <!-- <button class="btn btn-clear btn-icon" on:click={methods.selectDate}>
        <i class="zmdi zmdi-calendar" />
      </button> -->
          {#if searchMode}
            <button
              class="btn btn-clear btn-icon flex"
              on:click={methods.nextSearch}>
              <i class="zmdi zmdi-arrow-right " />
            </button>
          {:else}
            <button class="btn btn-clear btn-icon flex" on:click={methods.next}>
              <i class="zmdi zmdi-arrow-right" />
            </button>
          {/if}
        </div>
      </div>
      <!-- end toolbar div wrapper-->
    </NToolbar>

    <NToolbar>
      <div
        class="d-flex d-row justify-content-between align-items-center w-100
        search-bar">
        <input
          type="search"
          bind:this={searchInput}
          bind:value={state.searchTerm}
          on:keypress={methods.searchKeypress}
          placeholder="{Lang.t('general.search')}..."
          class="search-input" />
        {#if searchMode}
          <button
            class="btn btn-clear btn-sm btn-icon zmdi zmdi-search"
            on:click={methods.refreshSearch} />
          <button
            class="btn btn-clear btn-sm btn-icon zmdi zmdi-close"
            on:click={methods.clearSearch} />
        {/if}
      </div>
    </NToolbar>
  </header>
  <!-- end header-content header -->

  <main slot="content" class="page page-history">

    {#if loading}
      <div class="empty-notice">
        <Spinner />
      </div>
    {:else if state.showAllLocations}
      <NMap {locations} />
    {:else}
      <div class="container p-0">
        <!-- If no Logs found -->
        {#if logs.length === 0}
          {#if !searchMode}
            <div class="empty-notice">{Lang.t('history.no-records-found')}</div>
          {:else}
            <div class="empty-notice">
              {state.date.format('YYYY')} {Lang.t('history.no-records-found')}
            </div>
          {/if}
          <!-- If Logs and Not refreshing  -->
        {:else if !searchMode}
          <!-- Loop over logs -->
          {#each logs as log, i (log._id)}
            <LogItem
              {log}
              trackers={$TrackerStore}
              on:trackerClick={event => {
                methods.trackerTapped(event.detail.tracker, log);
              }}
              on:locationClick={event => {
                Interact.showLocations([log]);
              }}
              show24Hour={$UserStore.meta.is24Hour}
              on:moreClick={event => {
                Interact.logOptions(log).then(() => {});
              }} />
            <!-- Show the Log Item -->
          {/each}
        {:else if searchMode && searchLogs}
          {#each searchLogs as log, i (log._id)}
            <LogItem
              {log}
              fullDate={true}
              trackers={$TrackerStore}
              on:trackerClick={event => {
                methods.trackerTapped(event.detail.tracker, log);
              }}
              on:locationClick={event => {
                Interact.showLocations([log]);
              }}
              on:moreClick={event => {
                Interact.logOptions(log).then(() => {});
              }} />
            <!-- Show the Log Item -->
          {/each}
          {#if !searchLogs.length}
            <div class="gap" />
            <NItem title="No Results for {state.date.format('YYYY')}" />
          {/if}
          <div class="gap" />
          <NItem
            className="text-primary"
            on:click={methods.previousSearch}
            title="Search {state.date.subtract(1, 'year').format('YYYY')}..." />
          <div class="gap" />
        {:else if searchMode && !searchLogs}
          <div class="empty-notice">Search {state.date.format('YYYY')}</div>
        {/if}

      </div>
    {/if}
    {#if locations.length}
      {#if !state.showAllLocations}
        <div
          class="mini-map"
          on:click={() => {
            state.showAllLocations = !state.showAllLocations;
          }}>
          <NMap {locations} />
        </div>
      {:else}
        <div
          class="mini-map opened"
          on:click={() => {
            state.showAllLocations = !state.showAllLocations;
          }}>
          {Lang.t('history.close-map', 'Close Map')}
        </div>
      {/if}
    {/if}
  </main>
  <!-- end header-content content -->

</AppLayout>

{#if state.location.lat}
  <NModal show={true} title={state.location.name || 'Location'}>
    <NMap locations={[state.location]} />
    <button
      class="btn btn-lg btn-primary btn-block mb-0"
      on:click={methods.clearLocation}
      slot="footer">
      Close
    </button>
  </NModal>
{/if}

{#if local.showDatePicker}
  <NModal show={true} title={'Select a Date'}>
    <NDatePicker
      on:change={event => {
        local.datePickerValue = event.detail;
      }} />
    <button
      class="btn btn-lg btn-light btn-block m-0 mr-1"
      on:click={() => {
        local.showDatePicker = false;
      }}
      slot="footer">
      Close
    </button>
    <button
      class="btn btn-lg btn-primary btn-block m-0 ml-1"
      on:click={() => {
        local.showDatePicker = false;
        methods.goto(dayjs(local.datePickerValue));
      }}
      slot="footer">
      Go
    </button>
  </NModal>
{/if}

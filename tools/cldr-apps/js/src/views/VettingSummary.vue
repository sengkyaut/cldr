<template>
  <div v-if="!canUseSummary">
    <p>{{ accessDenied }}</p>
  </div>
  <div v-else>
    <h1 v-if="heading">{{ heading }}</h1>
    <p v-if="whenReceived">Received: {{ whenReceived }}</p>
    <p v-if="helpMessage">{{ helpMessage }}</p>
    <span v-html="output"></span>
    <hr />
    <p v-if="status">Current Status: {{ status }}</p>
    <p v-if="message">
      <span v-html="message"></span>
    </p>
    <div v-if="percent" class="summaryPercent">
      <a-progress :percent="percent" />
    </div>
    <p>
      <button v-if="canStop()" @click="stop()">Stop</button>
      <button
        v-else
        title="Create a new Priority Items Summary (not a snapshot)"
        @click="start()"
      >
        Create New Summary
      </button>
    </p>
    <section v-if="snapshotsAreReady" class="snapSection">
      <h2 class="snapHeading">Snapshots</h2>
      <p>
        <button
          v-if="canCreateSnapshots"
          title="Create a new snapshot of the Priority Items Summary"
          @click="createSnapshot"
        >
          Create New Snapshot
        </button>
        <span v-if="snapshotArray">
          <span v-for="snapshotId of snapshotArray" :key="snapshotId">
            <button
              :title="getSnapshotHover(snapshotId)"
              @click="showSnapshot(snapshotId)"
            >
              {{ humanizeSnapId(snapshotId) }}
            </button>
          </span>
        </span>
      </p>
    </section>

    <hr />

    <section class="snapSection">
      <h2 class="snapHeading">Report Status</h2>
      <a-spin size="small" v-if="reportLoad && !reports" />
      <button @click="fetchReports">Load</button>
      <button @click="downloadReports">Download</button>
      <table class="reportTable" v-if="reports">
        <thead>
          <tr>
            <th>Locale</th>
            <th v-for="type of reports.types" :key="type">{{ humanizeReport(type) }}</th>
            <th>Overall</th>
          </tr>
        </thead>
        <tbody>
          <tr
            v-for="locale of Object.keys(reports.byLocale).sort()"
            :key="locale"
          >
            <td>
              <a class="locale" :href="linkToLocale(locale)">{{ locale }}—{{ humanizeLocale(locale) }}</a>
            </td>
            <td :title="humanizeReport(type)" v-for="type of reports.types" :key="type">
                <div class="d-dr-status statuscell" :class="'d-dr-' + reports.byLocale[locale].byReport[type].status">&nbsp;</div>
                {{ reports.byLocale[locale].byReport[type].status }}:
                {{ reports.byLocale[locale].byReport[type].acceptability }}
            </td>
            <td>
              <span
                class="reportEntry"
                v-for="kind of ['acceptable', 'unacceptable', 'totalVoters']"
                :key="kind"
              >
                <i
                  v-if="reports.byLocale[locale][kind]"
                  :class="reportClass(kind)"
                  >&nbsp;</i
                >
                {{ kind }}={{ reports.byLocale[locale][kind] }}
              </span>
            </td>
          </tr>
        </tbody>
      </table>
    </section>
  </div>
</template>

<script>
import * as cldrLoad from "../esm/cldrLoad.js";
import * as cldrPriorityItems from "../esm/cldrPriorityItems.js";
import * as cldrText from "../esm/cldrText.js";
import * as cldrReport from "../esm/cldrReport.js";

export default {
  data() {
    return {
      accessDenied: null,
      canCreateSnapshots: false,
      canUseSnapshots: false,
      canUseSummary: false,
      heading: null,
      helpMessage: null,
      message: null,
      output: null,
      percent: 0,
      snapshotArray: null,
      snapshotsAreReady: false,
      status: null,
      whenReceived: null,
      reports: null,
      reportLoad: false,
    };
  },
  created() {
    cldrPriorityItems.viewCreated(this.setData, this.setSnapshots);
    this.canUseSummary = cldrPriorityItems.canUseSummary();
    this.canUseSnapshots = cldrPriorityItems.canUseSnapshots();
    this.canCreateSnapshots = cldrPriorityItems.canCreateSnapshots();
    if (!this.canUseSummary) {
      this.accessDenied = cldrText.get("summary_access_denied");
    }
  },

  methods: {
    start() {
      cldrPriorityItems.start();
    },

    stop() {
      cldrPriorityItems.stop();
    },

    showSnapshot(snapshotId) {
      cldrPriorityItems.showSnapshot(snapshotId);
    },

    createSnapshot() {
      cldrPriorityItems.createSnapshot();
    },

    setData(data) {
      this.message = data.message;
      this.percent = data.percent;
      if (data.status) {
        this.status = data.status;
      }
      if (data.output) {
        this.output = data.output;
        this.heading = this.makeHeading(data.snapshotId);
        this.helpMessage = this.makeHelp(data.snapshotId);
        this.whenReceived = this.makeWhenReceived(data.snapshotId);
        this.snapshotsAreReady = this.canUseSnapshots && this.snapshotArray;
      }
    },

    makeWhenReceived(snapshotId) {
      if (!this.output) {
        return null;
      }
      if (cldrPriorityItems.snapshotIdIsValid(snapshotId)) {
        return null;
      } else {
        // only show "when received" if it's not a snapshot
        return new Date().toString();
      }
    },

    makeHeading(snapshotId) {
      if (!this.output) {
        return null;
      }
      if (cldrPriorityItems.snapshotIdIsValid(snapshotId)) {
        return "Snapshot " + this.humanizeSnapId(snapshotId);
      } else if (this.canUseSnapshots) {
        return "Latest (not a snapshot)";
      } else {
        return "Latest";
      }
    },

    makeHelp(snapshotId) {
      if (!this.output) {
        return null;
      }
      let help = cldrText.get("summary_help") + " ";
      if (cldrPriorityItems.snapshotIdIsValid(snapshotId)) {
        help += cldrText.get("summary_coverage_neutral");
      } else {
        help += cldrText.get("summary_coverage_org_specific");
      }
      return help;
    },

    setSnapshots(snapshots) {
      this.snapshotArray = snapshots.array.sort().reverse();
      if (!this.output) {
        if (this.snapshotArray[0]) {
          // request this most recent snapshot from the back end
          // -- wait until get response to set snapshotsAreReady
          this.showSnapshot(this.snapshotArray[0]);
        } else {
          // no snapshots are available; we're ready to show the empty menu
          this.snapshotsAreReady = this.canUseSnapshots;
        }
      }
    },

    canStop() {
      return this.status === "WAITING" || this.status === "PROCESSING";
    },

    async fetchReports() {
      this.reportLoad = true;
      this.reports = await cldrReport.fetchAllReports();
    },

    humanizeReport(report) {
      return cldrReport.reportName(report);
    },

    humanizeLocale(locale) {
      return cldrLoad.getLocaleName(locale);
    },

    linkToLocale(locale) {
      return cldrLoad.linkToLocale(locale);
    },

    /**
     * Display a more user-friendly snapshot id
     *
     * @param id like “2022-05-16T08:15:25.083077935Z”
     * @return like “2022-05-16 08:15 UTC
     */
    humanizeSnapId(id) {
      if (!id.match(/^\d+\-\d+\-\d+T\d\d:\d\d:\d\d\..+Z$/)) {
        return id;
      }
      return id.replace("T", " ").replace(/:\d\d\..+Z/, " UTC");
    },

    getSnapshotHover(id) {
      const humanizedId = this.humanizeSnapId(id);
      const args = [humanizedId, id];
      return cldrText.sub("summary_snapshot_hover", args);
    },

    reportClass(kind) {
      if (kind === "unacceptable") {
        return cldrReport.reportClass(true, false);
      } else if (kind === "acceptable") {
        return cldrReport.reportClass(true, true);
      } else if (kind === "totalVoters") {
        return "totalVoters";
      } else {
        return cldrReport.reportClass(false, false);
      }
    },

    downloadReports() {
      return cldrReport.downloadAllReports("-");
    },
  },
};
</script>

<style scoped>
button {
  margin: 1ex;
}

.snapHeading {
  margin-top: 4px;
}

.snapSection {
  border: 2px solid gray;
  padding: 4px;
  background-color: #ccdfff; /* light blue */
}

.summaryPercent {
  margin: 1ex;
}

.reportTable th,
.reportTable td {
  padding: 0.5em;
  border-right: 2px solid gray;
}

.reportEntry {
  border-right: 1px solid gray;
  padding-right: 0.5em;
  display: table-cell;
}

.locale {
  background-color: beige;
  padding: 0.25em;
}
</style>

<template>
  <div class="table-component">
    <div class="table-component__table-wrapper">
      <table :class="fullTableClass">
          <thead :class="fullTableHeadClass">
            <tr>
              <table-column-header
                @click="changeSorting"
                v-for="column in columns"
                :key="column.show"
                :sort="sort"
                :column="column"
              ></table-column-header>
            </tr>
          </thead>
          <tbody :class="fullTableBodyClass">
          <table-row
            v-for="row in displayedRows"
            :key="row.vueTableComponentInternalRowId"
            :row="row"
            :columns="columns"
            @rowClick="emitRowClick"
          ></table-row>
          </tbody>
          <tfoot>
              <slot name="tfoot" :rows="rows"></slot>
          </tfoot>
      </table>
    </div>

    <div v-if="displayedRows.length === 0" class="table-component__message">
      {{ filterNoResults }}
    </div>

    <div style="display:none;">
      <slot></slot>
    </div>

    <template v-if="pagination && count">
      <slot name="pagination" v-bind:pagination="{ count, pageChange, paginationEllipsisClick }">
        <pagination
          :current-page="pagination.currentPage"
          :per-page="pagination.perPage"
          :count="count"
          @pageChange="pageChange"
          @ellipsisClick="paginationEllipsisClick"
        />
      </slot>
    </template>
  </div>
</template>

<script>
import Column from '../classes/Column';
import expiringStorage from '../expiring-storage';
import Row from '../classes/Row';
import TableColumnHeader from './TableColumnHeader';
import TableRow from './TableRow';
import settings from '../settings';
import Pagination from './Pagination';
import { classList, pick } from '../helpers';

export default {
  components: {
    TableColumnHeader,
    TableRow,
    Pagination,
  },

  props: {
    data: { default: () => [], type: [Array, Function] },
    pagination: { type: Object, default: undefined },

    showCaption: { default: true },

    sortBy: { default: '', type: String },
    sortOrder: { default: '', type: String },

    cacheKey: { default: null },
    cacheLifetime: { default: 5 },

    tableClass: { default: () => settings.tableClass },
    theadClass: { default: () => settings.theadClass },
    tbodyClass: { default: () => settings.tbodyClass },
    filterNoResults: { default: () => settings.filterNoResults },
  },

  data: () => ({
    columns: [],
    rows: [],
    sort: {
      fieldName: '',
      order: '',
    },
    count: undefined,

    localSettings: {},
  }),

  created() {
    this.sort.fieldName = this.sortBy;
    this.sort.order = this.sortOrder;

    this.restoreState();
  },

  async mounted() {
    const columnComponents = this.$slots.default
      .filter((column) => column.componentInstance)
      .map((column) => column.componentInstance);

    this.columns = columnComponents.map((column) => new Column(column));

    columnComponents.forEach((columnCom) => {
      Object.keys(columnCom.$options.props).forEach((prop) => columnCom.$watch(prop, () => {
        this.columns = columnComponents.map((column) => new Column(column));
      }));
    });

    await this.mapDataToRows();
  },

  watch: {
    data() {
      if (this.usesLocalData) {
        this.mapDataToRows();
      }
    },
  },

  computed: {
    fullTableClass() {
      return classList('table-component__table', this.tableClass);
    },

    fullTableHeadClass() {
      return classList('table-component__table__head', this.theadClass);
    },

    fullTableBodyClass() {
      return classList('table-component__table__body', this.tbodyClass);
    },

    ariaCaption() {
      if (this.sort.fieldName === '') {
        return 'Table not sorted';
      }

      return `Table sorted by ${this.sort.fieldName} ${
        this.sort.order === 'asc' ? '(ascending)' : '(descending)'}`;
    },

    usesLocalData() {
      return Array.isArray(this.data);
    },

    displayedRows() {
      let rows = this.sortedRows;

      if (this.usesLocalData && this.pagination) {
        const lastPage = this.pagination.currentPage - 1;
        const lastElementOfLastPageIndex = lastPage * this.pagination.perPage;
        rows = rows.slice(lastElementOfLastPageIndex, lastElementOfLastPageIndex + this.pagination.perPage);
      }

      return rows;
    },

    sortedRows() {
      if (!this.usesLocalData) {
        return this.rows;
      }

      if (this.sort.fieldName === '') {
        return this.rows;
      }

      if (this.columns.length === 0) {
        return this.rows;
      }

      const sortColumn = this.getColumn(this.sort.fieldName);

      if (!sortColumn) {
        return this.rows;
      }

      return this.cloneArray(this.rows).sort(sortColumn.getSortPredicate(this.sort.order, this.columns));
    },

    storageKey() {
      const storageWithCacheKey = `vue-table-component.${this.cacheKey}`;
      const storageWithoutCacheKey = `vue-table-component.${window.location.host}${window.location.pathname}${this.cacheKey}`;

      return this.cacheKey ? storageWithCacheKey : storageWithoutCacheKey;
    },
  },

  methods: {
    cloneArray(arrayToCopy) {
      return arrayToCopy.slice(0);
    },

    async pageChange(page) {
      this.pagination.currentPage = page;

      await this.mapDataToRows();
    },

    async mapDataToRows() {
      const data = this.usesLocalData ? this.prepareLocalData() : await this.fetchServerData();

      this.rows = data
        .map((rowData, rowIndex) => ({ ...rowData, vueTableComponentInternalRowId: rowIndex }))
        .map((rowData) => new Row(rowData, this.columns));
    },

    paginationEllipsisClick(event) {
      this.$emit('paginationEllipsisClick', event);
    },

    prepareLocalData() {
      this.count = this.data.length;

      return this.data;
    },

    async fetchServerData() {
      const page = (this.pagination && this.pagination.currentPage) || 1;

      const response = await this.data({
        sort: this.sort,
        page: page,
      });

      this.count = response.count;

      return response.data;
    },

    async refresh() {
      await this.mapDataToRows();
    },

    changeSorting(column) {
      if (this.sort.fieldName === column.show) {
        this.sort.order = this.sort.order === 'asc' ? 'desc' : 'asc';
      } else {
        this.sort.fieldName = column.show;
        this.sort.order = 'asc';
      }

      if (!this.usesLocalData) {
        this.mapDataToRows();
      }

      this.saveState();
    },

    getColumn(columnName) {
      return this.columns.find((column) => column.show === columnName);
    },

    saveState() {
      expiringStorage.set(this.storageKey, pick(this.$data, ['sort']), this.cacheLifetime);
    },

    restoreState() {
      const previousState = expiringStorage.get(this.storageKey);

      if (previousState === null) {
        return;
      }

      this.sort = previousState.sort;

      this.saveState();
    },

    emitRowClick(row) {
      this.$emit('rowClick', row);
      this.$emit('row-click', row);
    },
  },
};
</script>

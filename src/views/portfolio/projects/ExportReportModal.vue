<template>
  <b-modal
    id="exportReportModal"
    size="md"
    hide-header-close
    no-stacking
    :title="$t('message.export_report')"
    @show="onShow"
    @hide="resetValues()"
  >
    <b-form id="exportReportForm" @submit.prevent="exportReport">
      <b-form-group :label="$t('message.export_report_format')">
        <b-form-radio-group
          v-model="format"
          :options="formatOptions"
          name="export-report-format"
          button-variant="outline-primary"
          buttons
        />
      </b-form-group>

      <b-form-group :label="$t('message.export_report_sections')">
        <b-form-checkbox-group
          v-if="format === 'PDF'"
          v-model="pdfSections"
          :options="sectionOptions"
          stacked
        />
        <b-form-radio-group
          v-else
          v-model="csvSection"
          :options="csvSectionOptions"
          name="export-report-csv-section"
          stacked
        />
      </b-form-group>

      <b-form-checkbox v-model="includeSuppressed" switch>
        {{ $t('message.export_report_include_suppressed') }}
      </b-form-checkbox>
    </b-form>

    <template v-slot:modal-footer="{ cancel }">
      <b-button
        size="md"
        variant="secondary"
        :disabled="isExporting"
        @click="cancel()"
        >{{ $t('message.cancel') }}</b-button
      >
      <b-button
        type="submit"
        form="exportReportForm"
        size="md"
        variant="primary"
        :disabled="isSubmitButtonDisabled || isExporting"
      >
        <b-spinner v-if="isExporting" small class="mr-1"></b-spinner>
        {{ $t('message.export_report_submit') }}
      </b-button>
    </template>
  </b-modal>
</template>

<script>
import permissionsMixin from '@/mixins/permissionsMixin';

export default {
  name: 'ExportReportModal',
  mixins: [permissionsMixin],
  props: {
    uuid: String,
  },
  data() {
    return {
      format: 'PDF',
      pdfSections: [],
      csvSection: null,
      includeSuppressed: false,
      isExporting: false,
      formatOptions: [
        { text: 'PDF', value: 'PDF' },
        { text: 'CSV', value: 'CSV' },
      ],
    };
  },
  computed: {
    availableSections() {
      const sections = ['SUMMARY', 'COMPONENTS'];
      if (this.isPermitted(this.PERMISSIONS.VIEW_VULNERABILITY)) {
        sections.push('FINDINGS');
      }
      if (this.isPermitted(this.PERMISSIONS.VIEW_POLICY_VIOLATION)) {
        sections.push('POLICY_VIOLATIONS');
      }
      return sections;
    },
    sectionOptions() {
      return this.availableSections.map((section) => ({
        value: section,
        text: this.$t(`message.export_report_section_${section.toLowerCase()}`),
      }));
    },
    csvSectionOptions() {
      return this.sectionOptions.filter((option) => option.value !== 'SUMMARY');
    },
    isSubmitButtonDisabled() {
      if (this.format === 'CSV') {
        return !this.csvSection;
      }
      return this.pdfSections.length === 0;
    },
  },
  watch: {
    format() {
      this.pdfSections = [...this.availableSections];
      this.csvSection = null;
    },
  },
  methods: {
    onShow: function () {
      this.format = 'PDF';
      this.pdfSections = [...this.availableSections];
      this.csvSection = null;
      this.includeSuppressed = false;
    },
    resetValues: function () {
      this.isExporting = false;
    },
    exportReport: function () {
      if (this.isSubmitButtonDisabled || this.isExporting) {
        return;
      }
      const sections =
        this.format === 'CSV' ? [this.csvSection] : this.pdfSections;
      const params = new URLSearchParams();
      params.append('format', this.format);
      sections.forEach((section) => params.append('sections', section));
      params.append('include_suppressed', this.includeSuppressed);

      const url = `${this.$api.BASE_URL}/api/v2/projects/${this.uuid}/export`;
      this.isExporting = true;
      this.axios
        .request({
          responseType: 'blob',
          url: url,
          method: 'get',
          params: params,
        })
        .then((response) => {
          const objectUrl = window.URL.createObjectURL(
            new Blob([response.data]),
          );
          const link = document.createElement('a');
          link.href = objectUrl;
          let filename = `report.${this.format.toLowerCase()}`;
          const disposition = response.headers['content-disposition'];
          if (disposition && disposition.indexOf('attachment') !== -1) {
            const filenameRegex = /filename[^;=\n]*=((['"]).*?\2|[^;\n]*)/;
            const matches = filenameRegex.exec(disposition);
            if (matches != null && matches[1]) {
              filename = matches[1].replace(/['"]/g, '');
            }
          }
          link.setAttribute('download', filename);
          document.body.appendChild(link);
          link.click();
          this.$root.$emit('bv::hide::modal', 'exportReportModal');
        })
        .catch((error) => {
          this.handleExportError(error);
        })
        .finally(() => {
          this.isExporting = false;
        });
    },
    handleExportError: function (error) {
      const fallbackMessage = this.$t('message.export_report_error');
      if (error.response && error.response.data instanceof Blob) {
        error.response.data
          .text()
          .then((text) => {
            try {
              const problem = JSON.parse(text);
              this.$toastr.e(problem.detail || fallbackMessage);
            } catch {
              this.$toastr.e(fallbackMessage);
            }
          })
          .catch(() => {
            this.$toastr.e(fallbackMessage);
          });
      } else {
        this.$toastr.e(fallbackMessage);
      }
    },
  },
};
</script>

<template>
  <div :class="classes">
    <h3 class="text-90 mb-4">{{ field.name }}</h3>
    <note-input
      v-if="field.addingNotesEnabled"
      v-model="note"
      @onSubmit="createNote"
      :loading="loading"
      :fullWidth="field.fullWidth"
      :placeholder="field.placeholder || __('novaNotesField.defaultPlaceholder')"
      :trixEnabled="trixEnabled"
    />

    <note
      v-for="note in notesToShow"
      :fullWidth="field.fullWidth"
      :note="note"
      :key="note.id"
      :date-format="dateFormat"
      @onDeleteRequested="onNoteDeleteRequested"
    />

    <div class="flex justify-center mb-3 mt-3" v-if="hasMoreToShow">
      <span
        class="btn btn-default btn-primary leading-tight ml-2 px-3 text-sm text-center cursor-pointer"
        style="height: 24px; line-height: 24px"
        @click="maxToShow = void 0"
      >
        <!-- Cast to String to fix runtime crash in Nova 3.8.0 to 3.8.2 -->
        {{ __('novaNotesField.showMoreNotes', { hiddenNoteCount: String(notes.length - maxToShow) }) }}
      </span>
    </div>

    <delete-note-confirmation-modal
      v-if="showDeleteConfirmation"
      :show="showDeleteConfirmation"
      @close="showDeleteConfirmation = false"
      @confirm="deleteNote(noteToDelete)"
    />
  </div>
</template>

<script>
import Note from './Note';
import NoteInput from './NoteInput';
import DeleteNoteConfirmationModal from './DeleteNoteConfirmationModal';

export default {
  components: { Note, NoteInput, DeleteNoteConfirmationModal },
  props: ['resourceName', 'resourceId', 'field', 'extraClass'],
  data: () => ({
    note: '',
    loading: true,
    notes: [],
    showDeleteConfirmation: false,
    noteToDelete: void 0,
    maxToShow: 5,
    dateFormat: 'DD MMM YYYY HH:mm',
    trixEnabled: false,
    fullWidth: false,
  }),
  mounted() {
    this.fetchNotes();
  },
  computed: {
    params() {
      return {
        resourceId: this.resourceId,
        resourceName: this.resourceName,
      };
    },
    notesToShow() {
      if (this.maxToShow) return this.notes.slice(0, this.maxToShow);
      return this.notes;
    },
    hasMoreToShow() {
      return this.maxToShow && this.notes.length > this.maxToShow;
    },
    classes() {
      const defaultClasses = 'notes-field bg-20 px-4 pt-4 pb-2 rounded-b-lg overflow-hidden border-b dark:border-gray-700';
      return defaultClasses + (this.extraClass ? ` ${this.extraClass}` : '');
    },
  },
  methods: {
    async fetchNotes() {
      this.loading = true;

      const { data } = await Nova.request().get(`/nova-vendor/nova-notes/notes`, {
        params: this.params,
      });
      const { notes, date_format: dateFormat, trix_enabled: trixEnabled, full_width: fullWidth } = data;

      if (Array.isArray(notes)) this.notes = notes;
      this.dateFormat = dateFormat;
      this.trixEnabled = trixEnabled;
      this.fullWidth = fullWidth;

      this.loading = false;
    },
    async createNote() {
      this.loading = true;

      try {
        await Nova.request().post(`/nova-vendor/nova-notes/notes`, { note: this.note }, { params: this.params });
        await this.fetchNotes();
      } catch (e) {
        Nova.error(this.__('There was a problem submitting the form.'));
      }

      this.note = '';

      this.loading = false;
    },
    async deleteNote(note) {
      this.loading = true;

      try {
        await Nova.request().delete(`/nova-vendor/nova-notes/notes`, {
          params: this.params,
          data: { noteId: note.id },
        });
        await this.fetchNotes();
      } catch (e) {
        Nova.error('Unknown error when trying to delete the note.');
      }

      this.showDeleteConfirmation = false;
      this.loading = false;
    },
    onNoteDeleteRequested(note) {
      this.showDeleteConfirmation = true;
      this.noteToDelete = note;
    },
  },
};
</script>

<style lang="scss" scoped>
.notes-field:not(:last-child) {
  border-bottom-right-radius: 0;
  border-bottom-left-radius: 0;
}
</style>

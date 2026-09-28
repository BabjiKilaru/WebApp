import { CommonModule } from '@angular/common';
import { ChangeDetectorRef, Component, OnInit } from '@angular/core';
import { FormsModule } from '@angular/forms';
import { HttpClient } from '@angular/common/http';

import { Header } from '../header/header';

// One row per scheduled VISIT for the selected date (not deduped by
// patient) — mirrors pickpt.jsp's picker exactly, including coincident
// same-day visits showing as separate, explicitly-chosen rows.
export interface QuestionnaireScheduledVisit {
  visitId: number;
  patientId: number;
  firstName: string;
  lastName: string;
  mrn: string;
  visitType: string | null;
  visitSubType: string | null;
}

export interface QuestionnaireOption {
  id: string;
  name: string;
}

export interface QuestionnaireCategory {
  id: string;
  label: string;
  options: QuestionnaireOption[];
}

@Component({
  selector: 'app-questionnaire',
  standalone: true,
  imports: [ CommonModule, FormsModule, Header ],
  templateUrl: './questionnaire.html',
  styleUrl: './questionnaire.css'
})
export class Questionnaire implements OnInit {
  selectedDate = this.getTodayDate();
  selectedLanguage = 'English';
  selectedPatientId: number | null = null;
  // Resolved once, by primary key, when the user picks a specific
  // scheduled visit — not re-derived from a date at save time. See
  // history-visit-questionnaire.ts for the legacy-behavior citation.
  selectedVisitId: number | null = null;
  selectedQuestionnaireIds: string[] = [];
  errorMessage = '';

  scheduledVisits: QuestionnaireScheduledVisit[] = [];
  loadingVisits = false;
  visitsError = '';

  constructor(private http: HttpClient, private cdr: ChangeDetectorRef) {}

  ngOnInit(): void {
    this.loadScheduledVisits();
  }

  private loadScheduledVisits(): void {
    this.loadingVisits = true;
    this.visitsError = '';

    this.http.get<QuestionnaireScheduledVisit[]>(
      `/api/patients/scheduled`,
      { params: { date: this.selectedDate } }
    ).subscribe({
      next: visits => {
        this.scheduledVisits = visits;
        this.loadingVisits = false;
        this.cdr.markForCheck();
      },
      error: error => {
        console.error('Unable to load scheduled visits:', error);
        this.scheduledVisits = [];
        this.loadingVisits = false;
        this.visitsError = 'Unable to load patients for this date.';
        this.cdr.markForCheck();
      }
    });
  }

  // Mirrors the legacy picker (clinconn/lab/pickpt.jsp) exactly: 3 hardcoded
  // groups, 5 checkable items. There is no category concept in the legacy
  // DB — grouping was always hardcoded in the JSP markup, not data-driven.
  categories: QuestionnaireCategory[] = [
    {
      id: 'history',
      label: 'History questionnaire',
      options: [
        { id: 'history', name: 'History' }
      ]
    },
    {
      id: 'podsi',
      label: 'PODSI questionnaire',
      options: [
        { id: 'podci-parent', name: 'Child/Adolescent (Parent Reported)' },
        { id: 'podci-self', name: 'Adolescent (Self Reported)' }
      ]
    },
    {
      id: 'other',
      label: 'Other',
      options: [
        { id: 'sports', name: 'Sports' },
        { id: 'hip', name: 'Hip' }
      ]
    }
  ];

  get questionnaires(): QuestionnaireOption[] {
    return this.categories.flatMap(category => category.options);
  }

  languages: string[] = [
    'English',
    'Spanish'
  ];

  onDateChange(): void {
    this.errorMessage = '';
    this.selectedPatientId = null;
    this.selectedVisitId = null;
    this.selectedQuestionnaireIds = [];
    this.loadScheduledVisits();
  }

  // The one legacy-accurate resolution point: picking a row here resolves
  // both patientId and visitId together, by primary key, exactly once —
  // matching find_quest.jsp's currVisitID resolution off pickpt.jsp's list.
  selectVisit(visit: QuestionnaireScheduledVisit): void {
    this.selectedPatientId = visit.patientId;
    this.selectedVisitId = visit.visitId;
    this.selectedQuestionnaireIds = [];
    this.errorMessage = '';
  }

  isQuestionnaireSelected(questionnaireId: string): boolean {
    return this.selectedQuestionnaireIds.includes(questionnaireId);
  }

  toggleQuestionnaire(questionnaireId: string): void {
    if (!this.selectedPatientId) {
      return;
    }

    this.selectedQuestionnaireIds = this.isQuestionnaireSelected(questionnaireId)
      ? this.selectedQuestionnaireIds.filter(id => id !== questionnaireId)
      : [...this.selectedQuestionnaireIds, questionnaireId];

    this.errorMessage = '';
  }

  // Legacy (pickpt.jsp openQuest()) opens a blank, fullscreen, named popup
  // window synchronously, THEN targets the picker form's own submit into
  // it — find_quest.jsp (visit resolution + assembly) runs inside that
  // popup, not the original window. We mirror that split: this just opens
  // /questionnaire/fill in a popup with the picked visit + selections as
  // query params; QuestionnaireFill does the assemble call itself, same
  // as find_quest.jsp would. No logout — confirmed there isn't one in
  // legacy (see questionnaire-fill.ts).
  startQuestionnaire(): void {
    this.errorMessage = '';
    if (!this.selectedPatientId || !this.selectedVisitId) {
      this.errorMessage = 'Please select a patient.';
      return;
    }
    if (this.selectedQuestionnaireIds.length === 0) {
      this.errorMessage = 'Please select at least one questionnaire.';
      return;
    }
    if (!this.selectedLanguage) {
      this.errorMessage = 'Please select a language.';
      return;
    }

    const visit = this.getSelectedVisit();
    const params = new URLSearchParams({
      patientId: String(this.selectedPatientId),
      visitId: String(this.selectedVisitId),
      selections: this.selectedQuestionnaireIds.join(','),
      language: this.selectedLanguage === 'Spanish' ? 'sp' : 'en',
      patientName: visit ? `${visit.firstName} ${visit.lastName}`.trim() : '',
      firstName: visit ? visit.firstName.trim() : ''
    });

    const popup = this.openFullscreenPopup(`/questionnaire/fill?${params.toString()}`);
    if (!popup) {
      this.errorMessage = 'Please allow pop-ups for this site to open the questionnaire.';
    }
  }

  // Matches pickpt.jsp's openQuest() sizing (screen.availHeight - 30,
  // screen.availWidth - 10, positioned at 0,0) — must be called
  // synchronously from a click handler or browsers will block it as an
  // unsolicited popup.
  private openFullscreenPopup(url: string): Window | null {
    const availHeight = (window.screen?.availHeight ?? window.innerHeight) - 30;
    const availWidth = (window.screen?.availWidth ?? window.innerWidth) - 10;
    const features = `left=0,top=0,width=${availWidth},height=${availHeight},scrollbars=yes,resizable=yes`;
    return window.open(url, 'questwin', features);
  }

  getSelectedVisit(): QuestionnaireScheduledVisit | undefined {
    return this.scheduledVisits.find(
      visit => visit.visitId === this.selectedVisitId
    );
  }

  private getTodayDate(): string {
    const today = new Date();
    const year = today.getFullYear();
    const month = String(
      today.getMonth() + 1
    ).padStart(2, '0');
    const day = String(
      today.getDate()
    ).padStart(2, '0');
    return `${year}-${month}-${day}`;
  }
}









<div class="questionnaire-page">
    <app-header></app-header>
    <main class="questionnaire-content">
        <div class="questionnaire-heading">
            <h1>
                Questionnaire
            </h1>
            <p>
                Select a patient and questionnaire to begin.
            </p>
        </div>

        <div class="questionnaire-layout">
            <section class="questionnaire-card patient-section">
                <div class="date-language-row">
                    <div class="control-group">
                        <label for="questionnaireDate">
                            Date
                        </label>
                        <input id="questionnaireDate" type="date" [(ngModel)]="selectedDate"
                            (ngModelChange)="onDateChange()" />
                    </div>

                    <div class="control-group">
                        <label for="questionnaireLanguage">
                            Language
                        </label>
                        <select id="questionnaireLanguage" [(ngModel)]="selectedLanguage">
                            @for (language of languages; track language) {
                            <option [value]="language">
                                {{ language }}
                            </option>
                            }
                        </select>
                    </div>
                </div>

                <div class="patient-heading">
                    <h3>
                        Select Patient
                    </h3>
                    <!-- @if (selectedPatientId) {
                    <span class="selected-label">
                        Patient Selected
                    </span>
                    } -->
                </div>

                @if (loadingVisits) {
                <div class="empty-patients">
                    Loading patients…
                </div>
                } @else if (visitsError) {
                <div class="empty-patients">
                    {{ visitsError }}
                </div>
                } @else if (scheduledVisits.length > 0) {
                <div class="patient-list">
                    @for (visit of scheduledVisits; track visit.visitId) {
                    <button type="button" class="patient-row" [class.selected]="selectedVisitId === visit.visitId"
                        (click)="selectVisit(visit)">
                        <span class="radio-indicator">
                            <span class="radio-inner" [class.selected]="selectedVisitId === visit.visitId"></span>
                        </span>
                        <span class="patient-name">
                            {{ visit.lastName }},
                            {{ visit.firstName }}
                            @if (visit.visitType) {
                            <span class="visit-type-badge">{{ visit.visitType }}</span>
                            }
                        </span>
                        <span class="patient-mrn">
                            MRN: {{ visit.mrn }}
                        </span>
                    </button>
                    }
                </div>
                } @else {
                <div class="empty-patients">
                    No patients are scheduled for the selected date.
                </div>
                }
            </section>

            <section class="questionnaire-card questionnaire-section">
                <div class="questionnaire-section-header">
                    <h3>
                        Select Questionnaire
                    </h3>
                    <p>
                        @if (!selectedPatientId) {
                        Select a patient before choosing a questionnaire.
                        } @else {
                        Select one or more questionnaires.
                        }
                    </p>
                </div>

                <div class="questionnaire-categories" [class.disabled]="!selectedPatientId">
                    @for (category of categories; track category.id) {
                    <div class="questionnaire-category">
                        <h4 class="category-label">{{ category.label }}</h4>
                        <div class="questionnaire-options">
                            @for (questionnaire of category.options; track questionnaire.id) {
                            <button type="button" class="questionnaire-option"
                                [class.selected]="isQuestionnaireSelected(questionnaire.id)"
                                [disabled]="!selectedPatientId" (click)="toggleQuestionnaire(questionnaire.id)">
                                <span class="checkbox-indicator">
                                    <span class="checkbox-check"
                                        [class.selected]="isQuestionnaireSelected(questionnaire.id)">✓</span>
                                </span>
                                <span class="questionnaire-name">
                                    {{ questionnaire.name }}
                                </span>
                            </button>
                            }
                        </div>
                    </div>
                    }
                </div>

                @if (errorMessage) {
                <div class="questionnaire-error">
                    {{ errorMessage }}
                </div>
                }

                <div class="questionnaire-actions">
                    <button type="button" class="start-btn"
                        [disabled]="!selectedPatientId || selectedQuestionnaireIds.length === 0"
                        (click)="startQuestionnaire()">
                        Start
                    </button>
                </div>
            </section>
        </div>
    </main>
</div>










:host {
  display: block;
  min-height: 100vh;
  background: #f7f9fa;
  color: #263238;
}

.questionnaire-page {
  min-height: 100vh;
  background: #f7f9fa;
}

.questionnaire-content {
  width: 100%;
  padding: 28px 30px 30px;
  box-sizing: border-box;
}

.questionnaire-heading {
  margin-bottom: 24px;
}

.questionnaire-heading h1 {
  margin: 0;
  font-size: 28px;
  font-weight: 700;
  color: #1f2937;
}

.questionnaire-heading p {
  margin: 6px 0 0;
  font-size: 13px;
  color: #6b7280;
}

.questionnaire-layout {
  display: grid;
  grid-template-columns: minmax(0, 1fr) minmax(0, 1fr);
  gap: 20px;
  align-items: stretch;
}

.questionnaire-card {
  background: #ffffff;
  border: 1px solid #e4e8eb;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.04);
  padding: 28px;
  box-sizing: border-box;
  min-height: 450px;
}

.date-language-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 18px;
  margin-bottom: 30px;
}

.control-group {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.control-group label {
  font-size: 14px;
  font-weight: 700;
  color: #1f2937;
}

.control-group input,
.control-group select {
  width: 100%;
  height: 40px;
  padding: 0 14px;
  box-sizing: border-box;
  border: 1px solid #d7dde2;
  border-radius: 6px;
  background: #ffffff;
  color: #263238;
  font-family: inherit;
  font-size: 13px;
}

.control-group input:focus,
.control-group select:focus {
  outline: none;
  border-color: #269c96;
  box-shadow: 0 0 0 3px rgba(38, 156, 150, 0.12);
}

.patient-heading {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  margin-bottom: 14px;
}

.patient-heading h3,
.questionnaire-section-header h3 {
  margin: 0;
  font-size: 20px;
  font-weight: 700;
  color: #1f2937;
}

.selected-label {
  padding: 5px 9px;
  border-radius: 999px;
  background: #e7f6f4;
  color: #16857f;
  font-size: 12px;
  font-weight: 600;
}

.patient-list {
  height: 240px;
  overflow-y: auto;
  overflow-x: hidden;
  border: 1px solid #d8e1e4;
  border-radius: 6px;
  background: #ffffff;
}

.patient-list::-webkit-scrollbar {
  width: 8px;
}

.patient-list::-webkit-scrollbar-track {
  background: #f1f3f4;
}

.patient-list::-webkit-scrollbar-thumb {
  background: #aeb8bd;
  border-radius: 10px;
}

.patient-list::-webkit-scrollbar-thumb:hover {
  background: #8d989d;
}

.patient-row {
  width: 100%;
  min-height: 40px;
  padding: 0 18px;
  display: grid;
  grid-template-columns: 28px minmax(0, 1fr) auto;
  align-items: center;
  gap: 12px;
  border: none;
  border-bottom: 1px solid #e4e8eb;
  background: #ffffff;
  color: #263238;
  cursor: pointer;
  text-align: left;
  font-family: inherit;
}

.patient-row:last-child {
  border-bottom: none;
}

.patient-row:hover {
  background: #f7fbfb;
}

.patient-row.selected {
  background: #eef8f7;
}

.patient-name {
  font-size: 13px;
  font-weight: 500;
  display: flex;
  align-items: center;
  gap: 8px;
}

.visit-type-badge {
  padding: 2px 8px;
  background: #eef2f3;
  border: 1px solid #d8e1e4;
  border-radius: 10px;
  color: #4b5563;
  font-size: 11px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.02em;
}

.patient-mrn {
  font-size: 13px;
  color: #667085;
  white-space: nowrap;
}

.radio-indicator {
  width: 21px;
  height: 21px;
  border: 2px solid #a9b3b8;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  box-sizing: border-box;
  flex-shrink: 0;
}

.patient-row.selected .radio-indicator,
.questionnaire-option.selected .radio-indicator {
  border-color: #269c96;
}

.radio-inner {
  width: 9px;
  height: 9px;
  border-radius: 50%;
}

.radio-inner.selected {
  background: #269c96;
}

.checkbox-indicator {
  width: 21px;
  height: 21px;
  border: 2px solid #a9b3b8;
  border-radius: 4px;
  display: flex;
  align-items: center;
  justify-content: center;
  box-sizing: border-box;
  flex-shrink: 0;
  transition: background 0.15s ease, border-color 0.15s ease;
}

.questionnaire-option.selected .checkbox-indicator {
  background: #269c96;
  border-color: #269c96;
}

.checkbox-check {
  color: #ffffff;
  font-size: 13px;
  line-height: 1;
  opacity: 0;
}

.checkbox-check.selected {
  opacity: 1;
}

.empty-patients {
  height: 180px;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
  border: 1px dashed #ced5da;
  border-radius: 6px;
  color: #7a858b;
  text-align: center;
  font-size: 14px;
}

.questionnaire-section {
  display: flex;
  flex-direction: column;
}

.questionnaire-section-header {
  margin-bottom: 22px;
}

.questionnaire-section-header p {
  margin: 8px 0 0;
  color: #6b7280;
  font-size: 14px;
}

.questionnaire-categories {
  display: flex;
  flex-direction: column;
  gap: 18px;
}

.questionnaire-categories.disabled {
  opacity: 0.58;
}

.category-label {
  margin: 0 0 8px;
  color: #1f2937;
  font-size: 13px;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.03em;
}

.questionnaire-options {
  border: 1px solid #d8e1e4;
  border-radius: 6px;
  overflow: hidden;
  background: #ffffff;
}

.questionnaire-option {
  width: 100%;
  min-height: 40px;
  padding: 0 18px;
  display: flex;
  align-items: center;
  gap: 14px;
  border: none;
  border-bottom: 1px solid #e4e8eb;
  background: #ffffff;
  color: #263238;
  cursor: pointer;
  text-align: left;
  font-family: inherit;
  font-size: 14px;
}

.questionnaire-option:last-child {
  border-bottom: none;
}

.questionnaire-option:not(:disabled):hover {
  background: #f7fbfb;
}

.questionnaire-option.selected {
  background: #eef8f7;
}

.questionnaire-option:disabled {
  cursor: not-allowed;
}

.questionnaire-name {
  line-height: 1.4;
}

.questionnaire-error {
  margin-top: 18px;
  padding: 10px 12px;
  border: 1px solid #efc5c5;
  border-radius: 5px;
  background: #fff4f4;
  color: #b42318;
  font-size: 13px;
}

.questionnaire-actions {
  margin-top: auto;
  display: flex;
  justify-content: flex-end;
  gap: 12px;
  padding-top: 28px;
}

.start-btn {
  min-width: 140px;
  height: 46px;
  padding: 0 24px;
  border: none;
  border-radius: 6px;
  background: #269c96;
  color: #ffffff;
  font-family: inherit;
  font-size: 15px;
  font-weight: 600;
  cursor: pointer;
}

.start-btn:hover:not(:disabled) {
  background: #218a85;
}

.start-btn:disabled {
  background: #b7d4d2;
  cursor: not-allowed;
}

.secondary-btn {
  min-width: 100px;
  height: 46px;
  padding: 0 20px;
  border: 1px solid #d7dde2;
  border-radius: 6px;
  background: #ffffff;
  color: #30363b;
  font-family: inherit;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
}

.secondary-btn:hover {
  background: #f7f9fa;
}

@media (max-width: 1000px) {
  .questionnaire-layout {
    grid-template-columns: 1fr;
  }
  .questionnaire-card {
    min-height: auto;
  }
  .questionnaire-actions {
    margin-top: 28px;
  }
}

@media (max-width: 650px) {
  .questionnaire-content {
    padding: 20px 16px 30px;
  }
  .questionnaire-card {
    padding: 20px;
  }
  .date-language-row {
    grid-template-columns: 1fr;
  }
  .patient-row {
    grid-template-columns: 26px 1fr;
  }
  .patient-mrn {
    grid-column: 2;
  }
  .start-btn {
    width: 100%;
  }
}













// Spanish text of the legacy questionnaire pages (clinconn/lab/quest/*.jsp), transcribed verbatim from
// the `if ("sp".equals(session.getAttribute("language")))` branches — including the legacy typos and
// unfinished strings ("continur", "Codiciones", the trailing "TODO"), because the goal is exact parity.
// See docs/legacy-questions/spanish-text.txt for the file/line of every entry.
//
// Keys are the English strings as rendered. A key may end in `#context` when the same English text has
// a different Spanish rendering on another page; the suffix is dropped for English.
// `[name]` stands for the patient's first name (currFName).
//
// Legacy has NO Spanish for: Hip (WOMAC/UCLA/Harris), Sports and UE History (their Struts pages always
// run with language.id = 1), nq-thanks ("End of Part One"), show-questionnaires (the overview list), and
// most PODCI variants (only the CH pages switch language, handled in podci-bank.ts).

export type QuestionnaireLanguage = 'en' | 'sp';

const SP: Record<string, string> = {
    // --- alerts ---------------------------------------------------------------------------------
    'Please fill in all answers before moving to the next question.':
        'Por favor, contesta todas las respuestas antes de continur a la próxima pregunta.',

    // --- begin-self / q-start / q-thanks / nq-start ---------------------------------------------
    'This section should be filled out by the patient.': 'Esta sección debe ser llenada por el paciente.',
    'Welcome to the duPont Hospital for Children Gait Lab':
        'Bienvenido al laboratorio de andar (Gait Lab) de duPont Hospital para los hijos.',
    'In order to best understand our patients, we need your help with some history about [name].':
        'Para entender mejor a nuestros pacientes , necesitamos su ayuda con un poco de historia sobre [name].',
    'In order to best understand our patients, we need your help with some history about [name].#nq':
        'Para entender mejor a nuestros pacientes, necesitamos su ayuda con un poco de historia sobre [name].',
    "If you have any questions while completing the questionnaire, please don't hesitate to ask a member of our staff for help.":
        'Si usted tiene preguntas al completar el cuestionario, por favor no dude en preguntar a un miembro de nuestro personal en busca de ayuda.',
    'These questions will be asked on each visit so we can stay up-to-date with you.':
        'Estas preguntas se harán en cada visita para que podamos estar al día con usted.',
    'Since this is your first time, there are two sections of history questions. This is the only time you will have to answer the questions in the first section.':
        'Dado que esta es la primera vez, hay dos secciones de preguntas de la historia. Esta es la única vez que tendrá que responder a las preguntas de la primera sección.',
    'Thank You': 'Gracias',
    'Thank You for Filling Out the Questionnaire. This information helps us get the big picture and provide the best possible care.':
        'Gracias por tomar el cuestionario.',
    'Please tell a member of our staff that you have finished.':
        'Por favor contar a un miembro del personal.',

    // --- nq-who / q-who -------------------------------------------------------------------------
    'Please tell us about you': 'Por favor, díganos sobre usted',
    'Your Name:': 'Tu nombre:',
    'What is your relationship to [name]?': '¿Cuál es su relación con [name]?',
    'Choose from the list': 'Escoge de la lista',

    // --- nq-birth / nq-walking / nq-learning ----------------------------------------------------
    "[name]'s Birth": "[name]'s Nacimiento",
    'How premature was [name]?': '¿Cómo prematura fue [name]?',
    'How much extended hospitalization did [name] require after birth?':
        '¿Cuánto hospitalización prolongada se requiería [name] después del nacimiento?',
    "[name]'s Walking History": 'la historia de andar de [name]',
    'What age did [name] first walk?': '¿Cuándo empezó a andar [name]?',
    'What age did [name] start talking?': '¿En qué edad [name] empezó de hablar?',
    'Learning': 'Aprendizaje',
    'How do you think [name] is able to learn compared to other children of the same age?':
        '¿Cómo cree que [name] puede aprender en comparación con otros niños de la misma edad?',

    // --- q-history / q-add_history --------------------------------------------------------------
    'Health History': 'la historia de la salud',
    "Here is what we show for [name]'s Health History:": 'Aquí es lo que mostramos para la historia de salud sobre [name]:',
    'Problem': 'Problema',
    'Age': 'edad',
    'Has [name] had any of the following problems?': '¿Tiene [name] unos de los siguientes problemas?',
    'Has [name] had any of the following problems that are not included in the chart above?':
        '¿Tiene [name] unos de los siguientes problemas que no están incluídos en la lista arriba?',
    'Add to the Health History': 'añadir a la historia de la salud',
    'Please select the problem and age that [name] was when it happened. If you are not sure, select "Not Sure" in the Age list.':
        "Por favor, indicar el problema y la edad que [name] tuvo cuado occurió. Si no estás seguro, indicar 'No estoy seguro.' en la lista de edad.",
    'Select One Problem': 'Seleccionar un problema',
    'Add This To Record': 'añadir este a la ficha',

    // --- q-conditions ---------------------------------------------------------------------------
    'Health Conditions': 'Codiciones de salud',
    'Delete': 'Borrar',
    "Here is the information in [name]'s record to date. If the list is ok, move on to the next question. If you need to delete one of them from the list, click \"Delete\" in that row.":
        'Aquí es lo que mostramos para la historia de salud sobre [name] ahora. Si esta lista está bíen, continua hasta la próxima pregunta. Si necesita Ud. borrar un elemento de la lista, empuja el "Borrar" en esa fila.',
    'Condition': 'Condiciones',
    'Does [name] have any conditions that are listed below?': '¿Tiene [name] unas condiciones de la lista abajo?',
    'Does [name] have any other conditions that are listed below?': '¿Tiene [name] otras condiciones de la lista abajo?',
    'If so, please select them then move to the next question.': 'Si el niño tiene las condiciones, eligui TODO',
    'Confirm Condition Deletion': 'confirmar la eliminación de la condición',
    'Are you sure you want to remove the following entry from your list?':
        '¿Ud. está seguro de que quiere remover los siguientes de la lista?',
    'Yes - Delete this from my List': 'Sí, remover esto de la lista',
    'Do Not Delete this Item': 'No borrar este artículo',

    // --- shared list-page instructions (devices / gait concerns / seizures / pain) --------------
    'Here is the information you\'ve given so far today. If the list is ok, move on to the next question. If you need to delete one of them from the list, click "Delete" in that row.':
        'Aquí está la información que ha dado hasta el momento actual. Si la lista está bien, pasar a la siguiente pregunta. Si necesita eliminar uno de ellos en la lista, haga clic en "Borrar" en la fila.',
    'Here is the information you\'ve given so far today. If the list is ok, move on to the next question. If you need to delete one of them from the list, click "Delete" in that row.#concerns':
        'Aquí está la información que ha dado hasta el momento actual . Si la lista está bien, pasar a la siguiente pregunta. Si necesita eliminar uno de ellos en la lista , haga clic en "Borrar" en la fila.',
    'Here is the information you\'ve given so far today. If the list is ok, move on to the next question. If you need to change or delete one of them, click "Update" in that row.':
        'Aquí está la información que ha dado hasta el momento actual. Si la lista está bien, pasar a la siguiente pregunta. Si necesita eliminar uno de ellos en la lista, haga clic en "Actualizar" en la fila.',

    // --- botox ----------------------------------------------------------------------------------
    'Botox Shot Information': 'Información sobre las inyecciones de Botox',
    'A botox shot is a shot placed into tight muscles to loosen them.':
        'Una inyección de Botox está puesto en los músculos apretados para evitar que se peguen.',
    'Below is the information we have on file regarding past Botox injections.':
        'Abajo es la infomación que tenemos sobre las TODO',
    'No Shots on Record': 'No historía de inyecciones',
    'Where': '¿Dónde?',
    'Date': 'Fecha',
    'Side': 'Lado',
    'Hospital/Clinic': 'Hospital/clínica',
    'Hospital/Clinic#form': 'Hospital/Clinica',
    'Physician': 'Médico',
    'Please select an option:': 'Por favor, elige una opción:',
    'Add a New Shot': 'añadir una nueva inyección',
    'Botox Shot Information - Add a New Shot': 'Información sobre las inyecciones de Botox- Añadir una nueva inyección',
    'Body Location': 'el lugar en el cuerpo',
    'Left': 'izquierda',
    'Right': 'derecha',
    'Both': 'ambos',
    'Cancel Add Shot': 'borar esta nueva inyección',

    // --- devices --------------------------------------------------------------------------------
    'Devices and Braces': 'Dispositivos de asistencia',
    'Device': 'Dispositivo',
    'Does [name] currently use any devices? If so, please select them then move to the next question.':
        '[name] está utilizando ahora algunos dispositivos? Si es así, por favor seleccione y luego pasar a la siguiente pregunta',
    'Does [name] currently use any other devices? If so, please select them then move to the next question.':
        '[name] está utilizando ahora algunos otros dispositivos? Si es así, por favor seleccione y luego pasar a la siguiente pregunta',
    'Both#side': 'Ambos',
    'R only': 'Derecho',
    'L only': 'Izquierda',
    'Click here to enter a device that is not in the list':
        'Haga clic aquí para indicar un dispositivo qué no está en la lista',
    'Device Entry': 'Dispositivo',
    'Enter name of Device:': 'Nombre del dispositivo:',
    'Add this Device': 'añadir este dispositivo',
    'Click here to return to the previous page without adding a new device':
        'Haga clic aquí que volver a pagina previa sin añadir un dispositivo nuevo',
    'Confirm Device Deletion': 'confirmar la eliminación del dispositivo',
    'Yes - Delete this from my List#dot': 'Sí, remover esto de la lista.',

    // --- seizure medications --------------------------------------------------------------------
    'Seizure Medications': 'Los tratamientos para las convulsiones',
    'Seizure Medications#entry': 'Medicamento para las convulsiones',
    'Medication Name': 'Medicamentos',
    'Does [name] currently use any seizure medications? If so, please select them then move to the next question.':
        '[name] está utilizando ahora algunos medicamentos para las convulsiones? Si es así, por favor seleccione y luego pasar a la siguiente pregunta',
    'Does [name] currently use any other seizure medications? If so, please select them then move to the next question.':
        '[name] está utilizando ahora algunos otros medicamentos para las convulsiones? Si es así, por favor seleccione y luego pasar a la siguiente pregunta',
    'Click here to enter a medication that is not in the list':
        'Haga clic aquí para indicar un medicamento qué no está en la lista',
    'Enter name of Medication:': 'Medicamento:',
    'Add this Medication': 'añadir este medicamento',
    'Click here to return to the previous page without adding a new medication':
        'Haga clic aquí que volver a pagina previa sin añadir un medicamento nuevo',
    'Confirm Medication Deletion': 'confirmar la eliminación del medicamento',

    // --- pain -----------------------------------------------------------------------------------
    'Pain Information': 'Dolor',
    'Location of Pain': 'Lugar de Dolor',
    'Does [name] have pain in any places listed? If so, please select where the pain is located then move to the next question.':
        'Tiene [name] dolor en algunas lugares? Si es así, por favor seleccione y luego pasar a la siguiente pregunta',
    'Does [name] have pain in any other places listed? If so, please select where the pain is located then move to the next question.':
        'Tiene [name] dolor en algunas otras lugares? Si es así, por favor seleccione y luego pasar a la siguiente pregunta',
    'Upper': 'Superior',
    'Lower': 'Baja',
    'Left#pain': 'Izquierdo',
    'Right#pain': 'Derecho',
    'Both#pain': 'Ambos',
    'None': 'Ninguna',
    'Update': 'Actualizar',

    // --- physical therapy -----------------------------------------------------------------------
    'Physical Therapy Services': 'Terapia física',
    'If [name] receives any therapy, please select the frequency of PT and/or OT below:':
        'Si [name] somete alguna terapia, por favor indicar la frecuencia abajo:',
    'PHYSICAL THERAPY': 'TERAPIA FÍSICA',
    'OCCUPATIONAL THERAPY': 'TERAPIA OCUPACIONAL',
    'Hospital': 'Hospital',
    'Clinic': 'Clínica',
    'School': 'Escuela',
    'Home': 'Hogar',

    // --- followup -------------------------------------------------------------------------------
    'Followup Date': 'Fecha de seguimiento',
    'If you know the date of the followup appointment with your doctor, please enter it below:':
        'Si sabe la fecha de la cita de seguimiento con su médico , por favor indica abajo:',
    '(for example, 2/5/2004 or Sept 2003)': 'por ejemplo 5 febrero 2004 o septiembre de 2003',

    // --- gait pages -----------------------------------------------------------------------------
    'Walking Support': 'Apoyo a caminar',
    "Which currently describes [name]'s need for support while walking?":
        '¿Cuál situación describe la necesidad de [name] para el apoyo a caminar ahora?',
    'How does [name] move around for short distances in the house?': '¿Cómo [name] se mueve para distancias cortas en la casa?',
    'How does [name] move around in and between classes at school?': '¿Cómo [name] se mueve entre las clases en la escuela?',
    'How does [name] move around for long distances such as the shopping center?':
        '¿Cómo [name] se mueve para distancias largas en lugares como un centro comercial?',
    'Changes in Walking': 'Los cambios en andar',
    "In the last 6-12 months (or since last visit), how has [name]'s ability to walk changed?":
        '¿Dentro las últimas 6-12 meses (o la última visita) cómo la capacidad de andar de [name] ha cambiado?',
    'No Change': 'no cambio',
    'Walks Much Better': 'Anda mucho más mejor',
    'Walks a Little Better': 'Anda mejor',
    'Walks a Little Worse': 'Anda peor',
    'Walks Much Worse': 'Anda mucho más peor',
    'Current Gait Concerns': 'Preocupaciones sobre Gait que tiene ahora',
    'Gait Concern': 'Preocupaciones sobre Gait que tiene ahora',
    'Do you currently have any concerns listed below? If so, please select them then move to the next question.':
        '¿Tiene preocupaciónes que están en la lista abajo? Si es así , por favor seleccione y luego pasar a la siguiente pregunta.',
    'Do you currently have any other concerns listed below? If so, please select them then move to the next question.':
        '¿Tiene otras preocupaciónes que están en la lista abajo? Si es así , por favor seleccione y luego pasar a la siguiente pregunta.',
    'Confirm Gait Concern Deletion': 'Confirm Gait Concern Deletion', // English-only in legacy

    // --- q-concerns (Spanish has no heading, one combined prompt) -------------------------------
    'Any Concerns?': '',
    'Please type in any other concerns you have today:': '¿Tiene otras preocupaciónes ahora?'
};

// Confirmation pages (q-*_update.jsp) carry English-only lines for some strings; this is the legacy rule.
const ENGLISH_ONLY = new Set(['Confirm Gait Concern Deletion']);

export function translate(key: string, language: QuestionnaireLanguage): string {
    const english = key.replace(/#\w+$/, '');
    if (language !== 'sp' || ENGLISH_ONLY.has(key)) {
        return english;
    }
    const spanish = SP[key];
    return spanish === undefined ? english : spanish;
}

export function fillName(text: string, firstName: string, fallback: string): string {
    return text.replace(/\[name\]/g, firstName || fallback);
}


















import { CommonModule } from '@angular/common';
import { ChangeDetectorRef, Component, EventEmitter, Input, OnChanges, Output } from '@angular/core';
import { FormsModule } from '@angular/forms';
import { HttpClient } from '@angular/common/http';

// Hip = WOMAC + UCLA Activity Score + Modified Harris Hip Score, migrated
// from the legacy hip-womac.jsp / hip-ucla.jsp / hip-harris.jsp flow.
// Question text, option labels, and Harris's per-option point values
// confirmed directly from source by the legacy session — see hip_score
// table (womac_01-24 / ucla_01-10 / harris_01-08), already exists, no
// schema changes needed.
//
// Takes visitId (resolved once at picker time), not a date — see
// history-visit-questionnaire.ts for the legacy-behavior citation
// (find_quest.jsp / currVisitID, never a (patient_id, date) lookup).

interface Scale5Item {
    key: string;
    text: string;
}

interface HarrisItem {
    key: string;
    label: string;
    options: { points: number; label: string }[];
}

@Component({
    selector: 'app-hip-questionnaire',
    standalone: true,
    imports: [CommonModule, FormsModule],
    templateUrl: './hip-questionnaire.html',
    styleUrl: './hip-questionnaire.css'
})
export class HipQuestionnaire implements OnChanges {
    @Input() patientId: number | null = null;
    @Input() visitId: number | null = null;
    @Input() hideActions = false;
    @Output() saveComplete = new EventEmitter<void>();
    @Output() saveFailed = new EventEmitter<string>();

    readonly womacScale = ['None', 'Mild', 'Moderate', 'Severe', 'Extreme'];

    readonly womacPain: Scale5Item[] = [
        { key: 'womac_01', text: 'Walking on a flat surface' },
        { key: 'womac_02', text: 'Going up and down stairs' },
        { key: 'womac_03', text: 'At night while in bed, pain disturbs your sleep' },
        { key: 'womac_04', text: 'Sitting or lying' },
        { key: 'womac_05', text: 'Standing upright' }
    ];

    readonly womacStiffness: Scale5Item[] = [
        { key: 'womac_06', text: 'How severe is your stiffness after first awakening in the morning?' },
        { key: 'womac_07', text: 'How severe is your stiffness after sitting, lying, or resting in the day?' }
    ];

    readonly womacFunction: Scale5Item[] = [
        { key: 'womac_08', text: 'Going down stairs' },
        { key: 'womac_09', text: 'Going up stairs' },
        { key: 'womac_10', text: 'Rising from sitting' },
        { key: 'womac_11', text: 'Standing' },
        { key: 'womac_12', text: 'Bending to the floor' },
        { key: 'womac_13', text: 'Walking on flat surfaces' },
        { key: 'womac_14', text: 'Getting in and out of a car, or on or off a bus' },
        { key: 'womac_15', text: 'Going shopping' },
        { key: 'womac_16', text: 'Putting on your socks or stockings' },
        { key: 'womac_17', text: 'Rising from the bed' },
        { key: 'womac_18', text: 'Taking off your socks or stockings' },
        { key: 'womac_19', text: 'Lying in bed' },
        { key: 'womac_20', text: 'Getting in or out of the bath' },
        { key: 'womac_21', text: 'Sitting' },
        { key: 'womac_22', text: 'Getting on or off the toilet' },
        { key: 'womac_23', text: 'Performance heavy domestic duties' },
        { key: 'womac_24', text: 'Performing light domestic duties' }
    ];

    readonly uclaOptions = [
        'Wholly Inactive, dependent on others, and can not leave residence',
        'Mostly Inactive or restricted to minimum activities of daily living',
        'Sometimes participates in mild activities, such as walking, limited housework and limited shopping',
        'Regularly Participates in mild activities',
        'Sometimes participates in moderate activities such as swimming or could do unlimited housework or shopping',
        'Regularly participates in moderate activities',
        'Regularly participates in active events such as bicycling',
        'Regularly participates in active events, such as golf or bowling',
        'Sometimes participates in impact sports such as jogging, tennis, skiing, acrobatics, ballet, heavy labor or backpacking',
        'Regularly participates in impact sports'
    ];

    readonly harrisItems: HarrisItem[] = [
        {
            key: 'harris_01', label: 'Pain', options: [
                { points: 44, label: 'None/ignores' },
                { points: 40, label: 'Slight, occasional, no compromise in activity' },
                { points: 30, label: 'Mild, no effect on ordinary activity, pain after activity, uses aspirin' },
                { points: 20, label: 'Moderate, tolerable, makes concessions, occasional codeine' },
                { points: 10, label: 'Marked, serious limitations' },
                { points: 0, label: 'Totally disabled' }
            ]
        },
        {
            key: 'harris_02', label: 'Limp', options: [
                { points: 11, label: 'None' },
                { points: 8, label: 'Slight' },
                { points: 5, label: 'Moderate' },
                { points: 0, label: 'Severe / Unable to walk' }
            ]
        },
        {
            key: 'harris_03', label: 'Support', options: [
                { points: 11, label: 'None' },
                { points: 7, label: 'Cane, long walks' },
                { points: 5, label: 'Cane, full time' },
                { points: 4, label: 'Crutch' },
                { points: 2, label: '2 canes' },
                { points: 1, label: '2 crutches' },
                { points: 0, label: 'Unable to walk' }
            ]
        },
        {
            key: 'harris_04', label: 'Distance Walked', options: [
                { points: 11, label: 'Unlimited' },
                { points: 8, label: '6 blocks' },
                { points: 5, label: '2-3 blocks' },
                { points: 2, label: 'Indoors only' },
                { points: 0, label: 'Bed and chair' }
            ]
        },
        {
            key: 'harris_05', label: 'Stairs', options: [
                { points: 4, label: 'Normally' },
                { points: 2, label: 'Normally with banister' },
                { points: 1, label: 'Any method' },
                { points: 0, label: 'Not able' }
            ]
        },
        {
            key: 'harris_06', label: 'Sock/Shoes', options: [
                { points: 4, label: 'With ease' },
                { points: 2, label: 'With difficulty' },
                { points: 0, label: 'Unable' }
            ]
        },
        {
            key: 'harris_07', label: 'Sitting', options: [
                { points: 5, label: 'Any chair, 1 hour' },
                { points: 3, label: 'High chair, 1/2 hour' },
                { points: 0, label: 'Unable to sit, 1/2 hour, any chair' }
            ]
        },
        {
            key: 'harris_08', label: 'Public Transportation', options: [
                { points: 1, label: 'Able to enter public transportation' },
                { points: 0, label: 'Unable to use public transportation' }
            ]
        }
    ];

    womac: Record<string, number | null> = {};
    ucla: number | null = null;
    harris: Record<string, number | null> = {};

    saving = false;
    saveError = '';
    saved = false;

    constructor(private http: HttpClient, private cdr: ChangeDetectorRef) {
        this.resetAnswers();
    }

    ngOnChanges(): void {
        this.resetAnswers();
        this.saved = false;
        this.saveError = '';
        this.preload();
    }

    // Legacy shows previously saved answers when the visit's questionnaire is reopened.
    private preload(): void {
        if (!this.patientId || !this.visitId) {
            return;
        }
        const visitId = this.visitId;
        this.http.get<{
            womac?: Record<string, number | null>;
            harris?: Record<string, number | null>;
            ucla?: number | null;
        }>(`/api/patients/${this.patientId}/hip-questionnaire/${visitId}`).subscribe({
            next: saved => {
                if (visitId !== this.visitId) return;
                for (const [key, value] of Object.entries(saved.womac ?? {})) {
                    if (key in this.womac && typeof value === 'number') this.womac[key] = value;
                }
                for (const [key, value] of Object.entries(saved.harris ?? {})) {
                    if (key in this.harris && typeof value === 'number') this.harris[key] = value;
                }
                if (typeof saved.ucla === 'number') this.ucla = saved.ucla;
                this.cdr.markForCheck();
            },
            // Nothing saved yet (or unreadable) — start blank like a first visit.
            error: () => {}
        });
    }

    private resetAnswers(): void {
        this.womac = {};
        for (const item of [...this.womacPain, ...this.womacStiffness, ...this.womacFunction]) {
            this.womac[item.key] = null;
        }
        this.ucla = null;
        this.harris = {};
        for (const item of this.harrisItems) {
            this.harris[item.key] = null;
        }
    }

    save(): void {
        if (!this.patientId || !this.visitId) {
            this.saveComplete.emit();
            return;
        }

        this.saving = true;
        this.saveError = '';

        this.http.post(
            `/api/patients/${this.patientId}/hip-questionnaire`,
            {
                visitId: this.visitId,
                womac: this.womac,
                ucla: this.ucla,
                harris: this.harris
            }
        ).subscribe({
            next: () => {
                this.saving = false;
                this.saved = true;
                this.cdr.markForCheck();
                this.saveComplete.emit();
            },
            error: error => {
                console.error('Unable to save Hip questionnaire:', error);
                this.saving = false;
                this.saveError = 'Unable to save. Please try again.';
                this.cdr.markForCheck();
                this.saveFailed.emit(this.saveError);
            }
        });
    }
}












<div class="hip-form">
    @if (saved) {
    <div class="submitted-banner">
        Hip questionnaire saved.
    </div>
    }

    <section class="hip-section">
        <h4>HOW MUCH PAIN HAVE YOU HAD IN YOUR HIP/KNEE IN THE LAST 2 DAYS?</h4>
        @for (item of womacPain; track item.key) {
        <div class="scale-row">
            <span class="scale-text">{{ item.text }}</span>
            <div class="scale-options">
                @for (label of womacScale; track label; let optionIndex = $index) {
                <label class="scale-option">
                    <input type="radio" [name]="item.key" [attr.name]="item.key" [value]="optionIndex" [(ngModel)]="womac[item.key]" />
                    {{ label }}
                </label>
                }
            </div>
        </div>
        }
    </section>

    <section class="hip-section">
        <h4>HOW MUCH STIFFNESS/DIFFICULTY MOVING YOUR HIP/KNEE HAVE YOU HAD IN THE LAST 2 DAYS?</h4>
        @for (item of womacStiffness; track item.key) {
        <div class="scale-row">
            <span class="scale-text">{{ item.text }}</span>
            <div class="scale-options">
                @for (label of womacScale; track label; let optionIndex = $index) {
                <label class="scale-option">
                    <input type="radio" [name]="item.key" [attr.name]="item.key" [value]="optionIndex" [(ngModel)]="womac[item.key]" />
                    {{ label }}
                </label>
                }
            </div>
        </div>
        }
    </section>

    <section class="hip-section">
        <h4>HOW MUCH DIFFICULTY HAVE YOU HAD WITH YOUR DAILY PHYSICAL ACTIVITIES OVER THE LAST 2 DAYS?</h4>
        @for (item of womacFunction; track item.key) {
        <div class="scale-row">
            <span class="scale-text">{{ item.text }}</span>
            <div class="scale-options">
                @for (label of womacScale; track label; let optionIndex = $index) {
                <label class="scale-option">
                    <input type="radio" [name]="item.key" [attr.name]="item.key" [value]="optionIndex" [(ngModel)]="womac[item.key]" />
                    {{ label }}
                </label>
                }
            </div>
        </div>
        }
    </section>

    <section class="hip-section">
        <h4>Please check one box that best describes current activity level.</h4>
        <div class="option-list">
            @for (label of uclaOptions; track label; let optionIndex = $index) {
            <label class="radio-option">
                <input type="radio" name="ucla" [value]="optionIndex + 1" [(ngModel)]="ucla" />
                {{ label }}
            </label>
            }
        </div>
    </section>

    <section class="hip-section">
        @for (item of harrisItems; track item.key) {
        <div class="harris-item">
            <span class="harris-label">{{ item.label }}</span>
            <div class="option-list">
                @for (option of item.options; track option.points) {
                <label class="radio-option">
                    <input type="radio" [name]="item.key" [attr.name]="item.key" [value]="option.points" [(ngModel)]="harris[item.key]" />
                    {{ option.label }}
                </label>
                }
            </div>
        </div>
        }
    </section>

    @if (saveError) {
    <div class="questionnaire-error">{{ saveError }}</div>
    }

    @if (!hideActions) {
    <div class="hip-actions">
        <button type="button" class="submit-btn" [disabled]="saving" (click)="save()">
            {{ saving ? 'Saving...' : (saved ? 'Save Again' : 'Save') }}
        </button>
    </div>
    }
</div>












:host {
    display: block;
}

.hip-form {
    display: flex;
    flex-direction: column;
    gap: 28px;
}

.submitted-banner {
    padding: 12px 16px;
    background: #eef6f4;
    border: 1px solid #a6d8cf;
    border-radius: 8px;
    color: #1f6b5e;
    font-size: 13px;
    font-weight: 500;
}

.hip-section {
    display: flex;
    flex-direction: column;
    gap: 10px;
}

.hip-section h4 {
    margin: 0 0 4px;
    padding-left: 12px;
    border-left: 3px solid #269c96;
    color: #1e293b;
    font-size: 15px;
    font-weight: 700;
}

.scale-row {
    display: flex;
    flex-direction: column;
    gap: 10px;
    padding: 14px 18px;
    background: #ffffff;
    border: 1px solid #e2e8f0;
    border-radius: 10px;
    transition: border-color 0.15s ease, box-shadow 0.15s ease;
}

.scale-row:hover {
    border-color: #cbd5e1;
    box-shadow: 0 1px 6px rgba(15, 23, 42, 0.05);
}

.scale-text {
    font-size: 14px;
    color: #1e293b;
    font-weight: 500;
}

.scale-options {
    display: flex;
    flex-wrap: wrap;
    gap: 6px 10px;
}

.scale-option {
    display: flex;
    align-items: center;
    gap: 6px;
    padding: 6px 12px;
    border: 1px solid #e2e8f0;
    border-radius: 999px;
    font-size: 12px;
    color: #374151;
    cursor: pointer;
    transition: background-color 0.12s ease, border-color 0.12s ease;
}

.scale-option:hover {
    background: #f8fafc;
}

.scale-option:has(input:checked) {
    background: #eef6f4;
    border-color: #269c96;
    color: #1f6b5e;
    font-weight: 600;
}

.scale-option input {
    width: 14px;
    height: 14px;
    accent-color: #269c96;
    cursor: pointer;
}


.option-list {
    display: flex;
    flex-direction: column;
    gap: 6px;
}

.radio-option {
    display: flex;
    align-items: flex-start;
    gap: 10px;
    padding: 9px 12px;
    border-radius: 8px;
    font-size: 13px;
    color: #374151;
    cursor: pointer;
    transition: background-color 0.12s ease;
}

.radio-option:hover {
    background: #f8fafc;
}

.radio-option:has(input:checked) {
    background: #eef6f4;
    color: #1f6b5e;
    font-weight: 500;
}

.radio-option input {
    width: 16px;
    height: 16px;
    margin-top: 1px;
    accent-color: #269c96;
    cursor: pointer;
    flex-shrink: 0;
}

.harris-item {
    display: flex;
    flex-direction: column;
    gap: 8px;
    padding: 14px 18px;
    background: #ffffff;
    border: 1px solid #e2e8f0;
    border-radius: 10px;
    transition: border-color 0.15s ease, box-shadow 0.15s ease;
}

.harris-item:hover {
    border-color: #cbd5e1;
    box-shadow: 0 1px 6px rgba(15, 23, 42, 0.05);
}

.harris-label {
    font-size: 14px;
    font-weight: 600;
    color: #1e293b;
}

.questionnaire-error {
    padding: 12px 14px;
    border: 1px solid #efc5c5;
    border-radius: 8px;
    background: #fff4f4;
    color: #b42318;
    font-size: 13px;
}

.hip-actions {
    display: flex;
    justify-content: flex-end;
}

.submit-btn {
    height: 40px;
    padding: 0 22px;
    color: #ffffff;
    background: #269c96;
    border: 1px solid #1f847f;
    border-radius: 8px;
    font-family: inherit;
    font-size: 14px;
    font-weight: 600;
    cursor: pointer;
    box-shadow: 0 1px 3px rgba(31, 132, 127, 0.25);
    transition: background-color 0.15s ease, box-shadow 0.15s ease;
}

.submit-btn:hover:not(:disabled) {
    background: #1f847f;
    box-shadow: 0 2px 6px rgba(31, 132, 127, 0.3);
}

.submit-btn:disabled {
    background: #b7d4d2;
    border-color: #b7d4d2;
    box-shadow: none;
    cursor: not-allowed;
}












import { CommonModule } from '@angular/common';
import { ChangeDetectorRef, Component, EventEmitter, Input, OnChanges, Output, SimpleChanges } from '@angular/core';
import { FormsModule } from '@angular/forms';
import { HttpClient } from '@angular/common/http';
import { forkJoin } from 'rxjs';

import { BaselineWho, REQUIRED_MESSAGE } from '../history-visit-questionnaire/history-visit-questionnaire';
import { fillName, QuestionnaireLanguage, translate } from '../legacy-i18n';

// NEW_HISTORY's first section: the four one-time patient pages nq-who / nq-birth / nq-walking /
// nq-learning (patients.hist_person, hist_rship, date_baseline, premature, birth_stay, age_walk,
// age_talk, learning). Legacy's NEW_HISTORY then continues with the general visit-history pages
// (everything in HISTORY except "who") — those are rendered by history-visit-questionnaire.ts.
//
// Behaviors verified against the JSPs:
//  - date_baseline is a HIDDEN field with no visible prompt: get_today() sets it to today's date when
//    the name field changes (an existing value is otherwise left untouched);
//  - every select is a required choice whose first row is a blank/"--" placeholder;
//  - nq-thanks.jsp copies the person/relationship into the visit's status_name/status_rship — the
//    values are emitted through `baselineChange` so the visit section can do that on save.
// Reuses the existing baseline endpoint (GET .../questionnaire-responses + .../options, PUT .../baseline);
// this scope has no visit_id (one row per patient).

interface Choice {
    value: string | number | null;
    label: string | null;
}

interface BaselineSection {
    fields: Record<string, string | number | null>;
    lists: Record<string, unknown[]>;
    version: string;
}

interface BaselinePage {
    baseline: BaselineSection;
}

// With ?language=… the lists come back as {value, label} (label = Spanish text or English fallback); the
// stored value is always the English one.
type LabelledOption = string | { value: string; label: string | null };

interface RawBaselineOptions {
    relationships: LabelledOption[];
    premature: LabelledOption[];
    birthStays: LabelledOption[];
    ages: LabelledOption[];
    learning: Choice[];
}

interface BaselineOptions {
    relationships: string[];
    premature: string[];
    birthStays: string[];
    ages: string[];
    learning: Choice[];
}

@Component({
    selector: 'app-history-questionnaire',
    standalone: true,
    imports: [CommonModule, FormsModule],
    templateUrl: './history-questionnaire.html',
    styleUrl: './history-questionnaire.css'
})
export class HistoryQuestionnaire implements OnChanges {
    @Input() patientId: number | null = null;
    @Input() firstName = '';
    @Input() hideActions = false;
    @Input() language: QuestionnaireLanguage = 'en';
    @Output() saveComplete = new EventEmitter<void>();
    @Output() saveFailed = new EventEmitter<string>();
    @Output() baselineChange = new EventEmitter<BaselineWho>();

    loading = false;
    loadError = '';
    saving = false;
    saveError = '';
    saved = false;

    section: BaselineSection | null = null;
    options: BaselineOptions | null = null;
    private labels: Record<string, string> = {};

    lab(kind: string, value: string | number | null | undefined): string {
        const key = String(value ?? '');
        return this.labels[`${kind}:${key}`] ?? key;
    }

    constructor(private http: HttpClient, private cdr: ChangeDetectorRef) {}

    // Only a change of what is being loaded resets the form — not e.g. the parent's live `copyWho`/`firstName`
    // updates, which would otherwise wipe whatever the user has already entered.
    ngOnChanges(changes: SimpleChanges): void {
        if (!(changes['patientId'] || changes['language'])) return;
        this.saved = false;
        this.saveError = '';
        this.section = null;
        this.options = null;
        this.load();
    }

    text(key: string): string {
        return fillName(translate(key, this.language), this.firstName, this.language === 'sp' ? 'su niño(a)' : 'your child');
    }

    private base(): string {
        return `/api/patients/${this.patientId}/questionnaire-responses`;
    }

    private load(): void {
        if (!this.patientId) {
            return;
        }

        this.loading = true;
        this.loadError = '';

        forkJoin({
            page: this.http.get<BaselinePage>(this.base()),
            options: this.http.get<RawBaselineOptions>(`${this.base()}/options`, { params: { language: this.language } })
        }).subscribe({
            next: ({ page, options }) => {
                this.section = page.baseline;
                this.labels = {};
                const values = (kind: string, list: LabelledOption[]) => list.map(item => {
                    if (typeof item === 'string') return item;
                    if (item.label) this.labels[`${kind}:${item.value}`] = item.label;
                    return item.value;
                });
                this.options = {
                    relationships: values('relationships', options.relationships ?? []),
                    premature: values('premature', options.premature ?? []),
                    birthStays: values('birthStays', options.birthStays ?? []),
                    ages: values('ages', options.ages ?? []),
                    learning: options.learning ?? []
                };
                this.loading = false;
                this.emitWho();
                this.cdr.markForCheck();
            },
            error: error => {
                console.error('Unable to load First Visit History:', error);
                this.loading = false;
                this.loadError = 'Unable to load this questionnaire.';
                this.cdr.markForCheck();
            }
        });
    }

    field(key: string): string | number | null {
        return this.section?.fields[key] ?? null;
    }

    setField(key: string, value: string | number | null): void {
        if (!this.section) return;
        this.section.fields[key] = value;
        if (key === 'historyRelationship') {
            this.emitWho();
        }
    }

    // nq-who.jsp: onChange on the name field runs get_today(), stamping the hidden date_baseline.
    setPerson(name: string): void {
        if (!this.section) return;
        this.section.fields['historyPerson'] = name;
        this.section.fields['baselineDate'] = this.today();
        this.emitWho();
    }

    private emitWho(): void {
        const fields = this.section?.fields;
        this.baselineChange.emit({
            person: (fields?.['historyPerson'] as string | null) ?? null,
            relationship: (fields?.['historyRelationship'] as string | null) ?? null
        });
    }

    private today(): string {
        const now = new Date();
        const month = String(now.getMonth() + 1).padStart(2, '0');
        const day = String(now.getDate()).padStart(2, '0');
        return `${now.getFullYear()}-${month}-${day}`;
    }

    // nq-who / nq-birth / nq-walking / nq-learning check_ok(): every field (text and selects) must be
    // filled before moving on.
    validate(): string | null {
        const fields = this.section?.fields;
        if (!fields) return null;
        const keys = ['historyPerson', 'historyRelationship', 'premature', 'birthStay', 'ageWalk', 'ageTalk', 'learning'];
        const blank = (value: unknown) => value === null || value === undefined || value === '' || value === '--';
        return keys.some(key => blank(fields[key])) ? translate(REQUIRED_MESSAGE, this.language) : null;
    }

    save(): void {
        if (!this.patientId || !this.section) {
            // Nothing loaded yet to save — count as done rather than
            // leaving an aggregate Save All stuck waiting on this section.
            this.saveComplete.emit();
            return;
        }

        // Legacy writes a blank text field as '' (never NULL); the numeric `learning` code is NULL when blank.
        for (const key of ['historyPerson', 'historyRelationship', 'premature', 'birthStay', 'ageWalk', 'ageTalk']) {
            if (this.section.fields[key] === null || this.section.fields[key] === undefined) this.section.fields[key] = '';
        }
        if (this.section.fields['learning'] === '' || this.section.fields['learning'] === undefined) {
            this.section.fields['learning'] = null;
        }

        this.saving = true;
        this.saveError = '';

        this.http.put<void>(`${this.base()}/baseline`, this.section).subscribe({
            next: () => {
                this.saving = false;
                this.saved = true;
                this.cdr.markForCheck();
                this.saveComplete.emit();
            },
            error: error => {
                console.error('Unable to save First Visit History:', error);
                this.saving = false;
                this.saveError = 'Unable to save. Please try again.';
                this.cdr.markForCheck();
                this.saveFailed.emit(this.saveError);
            }
        });
    }
}












<div class="history-form">
    @if (loading) {
    <div class="loading-text">Loading…</div>
    } @else if (loadError) {
    <div class="questionnaire-error">{{ loadError }}</div>
    } @else if (section && options) {

    @if (saved) {
    <div class="submitted-banner">
        First Visit History saved.
    </div>
    }
    @if (saveError) {
    <div class="questionnaire-error">{{ saveError }}</div>
    }

    <div class="history-body">
        <!-- nq-start.jsp -->
        <section class="history-page intro-page">
            <h2 class="page-heading">{{ text('Welcome to the duPont Hospital for Children Gait Lab') }}</h2>
            <p class="page-text">{{ text('In order to best understand our patients, we need your help with some history about [name].#nq') }}</p>
            <p class="page-text">{{ text("If you have any questions while completing the questionnaire, please don't hesitate to ask a member of our staff for help.") }}</p>
            <p class="page-text">{{ text('Since this is your first time, there are two sections of history questions. This is the only time you will have to answer the questions in the first section.') }}</p>
        </section>

        <!-- nq-who.jsp -->
        <section class="history-page">
            <h2 class="page-heading">{{ text('Please tell us about you') }}</h2>
            <div class="field-row">
                <label class="field-label" for="historyPerson">{{ text('Your Name:') }}</label>
                <input id="historyPerson" type="text" size="30"
                    [ngModel]="field('historyPerson')" (ngModelChange)="setPerson($event)" />
            </div>
            <div class="field-row">
                <label class="field-label" for="historyRelationship">
                    {{ text('What is your relationship to [name]?') }}
                    <span class="field-hint">{{ text('Choose from the list') }}</span>
                </label>
                <select id="historyRelationship"
                    [ngModel]="field('historyRelationship')" (ngModelChange)="setField('historyRelationship', $event)">
                    <option [ngValue]="null"></option>
                    @for (choice of options.relationships; track choice) {
                    <option [ngValue]="choice">{{ lab('relationships', choice) }}</option>
                    }
                </select>
            </div>
        </section>

        <!-- nq-birth.jsp -->
        <section class="history-page">
            <h2 class="page-heading">{{ text("[name]'s Birth") }}</h2>
            <div class="field-row">
                <label class="field-label" for="premature">{{ text('How premature was [name]?') }}</label>
                <select id="premature"
                    [ngModel]="field('premature')" (ngModelChange)="setField('premature', $event)">
                    <option [ngValue]="null"></option>
                    @for (choice of options.premature; track choice) {
                    <option [ngValue]="choice">{{ lab('premature', choice) }}</option>
                    }
                </select>
            </div>
            <div class="field-row">
                <label class="field-label" for="birthStay">{{ text('How much extended hospitalization did [name] require after birth?') }}</label>
                <select id="birthStay"
                    [ngModel]="field('birthStay')" (ngModelChange)="setField('birthStay', $event)">
                    <option [ngValue]="null"></option>
                    @for (choice of options.birthStays; track choice) {
                    <option [ngValue]="choice">{{ lab('birthStays', choice) }}</option>
                    }
                </select>
            </div>
        </section>

        <!-- nq-walking.jsp -->
        <section class="history-page">
            <h2 class="page-heading">{{ text("[name]'s Walking History") }}</h2>
            <div class="field-row">
                <label class="field-label" for="ageWalk">{{ text('What age did [name] first walk?') }}</label>
                <select id="ageWalk"
                    [ngModel]="field('ageWalk')" (ngModelChange)="setField('ageWalk', $event)">
                    <option [ngValue]="null"></option>
                    @for (choice of options.ages; track choice) {
                    <option [ngValue]="choice">{{ lab('ages', choice) }}</option>
                    }
                </select>
            </div>
            <div class="field-row">
                <label class="field-label" for="ageTalk">{{ text('What age did [name] start talking?') }}</label>
                <select id="ageTalk"
                    [ngModel]="field('ageTalk')" (ngModelChange)="setField('ageTalk', $event)">
                    <option [ngValue]="null"></option>
                    @for (choice of options.ages; track choice) {
                    <option [ngValue]="choice">{{ lab('ages', choice) }}</option>
                    }
                </select>
            </div>
        </section>

        <!-- nq-learning.jsp -->
        <section class="history-page">
            <h2 class="page-heading">{{ text('Learning') }}</h2>
            <div class="field-row">
                <label class="field-label" for="learning">{{ text('How do you think [name] is able to learn compared to other children of the same age?') }}</label>
                <select id="learning"
                    [ngModel]="field('learning')" (ngModelChange)="setField('learning', $event)">
                    <option [ngValue]="null"></option>
                    @for (choice of options.learning; track choice.value) {
                    <option [ngValue]="choice.value">{{ choice.label }}</option>
                    }
                </select>
            </div>
        </section>
    </div>

    @if (!hideActions) {
    <div class="history-actions">
        <button type="button" class="submit-btn" [disabled]="saving" (click)="save()">
            {{ saving ? 'Saving...' : (saved ? 'Save Again' : 'Save') }}
        </button>
    </div>
    }

    }
</div>

















:host {
    display: block;
}

.history-form {
    display: flex;
    flex-direction: column;
    gap: 14px;
}

.loading-text {
    color: #6b7280;
    font-size: 14px;
}

.submitted-banner {
    padding: 12px 16px;
    background: #eef6f4;
    border: 1px solid #a6d8cf;
    border-radius: 8px;
    color: #1f6b5e;
    font-size: 13px;
    font-weight: 500;
}

.questionnaire-error {
    padding: 12px 14px;
    border: 1px solid #efc5c5;
    border-radius: 8px;
    background: #fff4f4;
    color: #b42318;
    font-size: 13px;
}

.history-body {
    display: flex;
    flex-direction: column;
    gap: 14px;
}

.history-page {
    display: flex;
    flex-direction: column;
    gap: 14px;
    padding: 18px 20px;
    background: #ffffff;
    border: 1px solid #e2e8f0;
    border-radius: 10px;
}

.intro-page {
    background: #f8fafc;
}

.page-heading {
    margin: 0;
    color: #1e293b;
    font-size: 16px;
    font-weight: 700;
}

.page-text {
    margin: 0;
    color: #374151;
    font-size: 13px;
    line-height: 1.5;
}

.field-row {
    display: flex;
    flex-direction: column;
    gap: 6px;
}

.field-label {
    color: #1e293b;
    font-size: 13px;
    font-weight: 600;
}

.field-hint {
    margin-left: 8px;
    color: #6b7280;
    font-size: 12px;
    font-weight: 400;
    font-style: italic;
}

.history-page input[type="text"],
.history-page select {
    max-width: 360px;
    padding: 9px 12px;
    box-sizing: border-box;
    border: 1px solid #d7dde2;
    border-radius: 8px;
    background: #ffffff;
    color: #263238;
    font-family: inherit;
    font-size: 13px;
}

.history-page input:focus,
.history-page select:focus {
    outline: none;
    border-color: #269c96;
    box-shadow: 0 0 0 3px rgba(38, 156, 150, 0.12);
}

.history-actions {
    display: flex;
    justify-content: flex-end;
    padding-top: 4px;
}

.submit-btn {
    height: 40px;
    padding: 0 22px;
    color: #ffffff;
    background: #269c96;
    border: 1px solid #1f847f;
    border-radius: 8px;
    font-family: inherit;
    font-size: 14px;
    font-weight: 600;
    cursor: pointer;
    box-shadow: 0 1px 3px rgba(31, 132, 127, 0.25);
    transition: background-color 0.15s ease, box-shadow 0.15s ease;
}

.submit-btn:hover:not(:disabled) {
    background: #1f847f;
    box-shadow: 0 2px 6px rgba(31, 132, 127, 0.3);
}

.submit-btn:disabled {
    background: #b7d4d2;
    border-color: #b7d4d2;
    box-shadow: none;
    cursor: not-allowed;
}














import { CommonModule } from '@angular/common';
import { ChangeDetectorRef, Component, EventEmitter, Input, OnChanges, Output, SimpleChanges } from '@angular/core';
import { FormsModule } from '@angular/forms';
import { HttpClient } from '@angular/common/http';
import { forkJoin, from, of } from 'rxjs';
import { fillName, QuestionnaireLanguage, translate } from '../legacy-i18n';
import { catchError, concatMap, map, switchMap, tap, toArray } from 'rxjs/operators';

import {
    PatientHistoryData, HealthHistoryEntry, HistoryConditionOption,
    HealthConditionEntry, ConditionOption, BotoxHistory, SideOption
} from '../../patients/history/history';

// The legacy visit-history pages (clinconn/lab/quest/q-*.jsp), consolidated into one scrolling form.
// Which pages appear depends on which questionnaire codes were assembled — verified against
// QuestionnaireServiceImpl.getPosition():
//   HISTORY          = who, health history, conditions, seizures, botox, devices, pain, PT/OT, followup
//   NEW_HISTORY      = (first-visit pages, in history-questionnaire.ts) + the same list WITHOUT "who"
//   HISTORY_GAIT     = walking support, FMS 5/50/500, walking change, gait concerns
//   HISTORY_CONCERNS = concerns
// so `showWho` / `showGeneral` / `showGait` / `showConcerns` select the matching groups.
//
// Storage, verified page by page against the JSP save code:
//  - who: visits.status_name / status_rship (items.reportingPerson / relationship). On a first visit
//    legacy's nq-thanks.jsp copies the patient's baseline person/relationship into these instead of
//    asking again — see `copyWho`.
//  - health history / conditions / botox are patient-level (pt_id only, no visit_id) and save
//    immediately per action, like the legacy add pages; seizure meds, devices, pain, PT/OT, followup,
//    gait concerns, walking support/change, FMS and concerns are visit-level and save together with
//    save() (items/details/medications are deliberately saved from ONE component so two components
//    can't race to PUT-replace the same scope).
//  - devices: side is offered ONLY for Crutch and Cane (Both / R only / L only -> Both, R, L), else NULL.
//  - pain: one radio group per body part — Back: Upper / Lower / None, others: Left / Right / Both /
//    None (stored L/R/Both/Upper/Lower); None inserts no row.
//  - PT/OT: stores frequencies.code ('1'..'4'); the select has no blank option and lists None first
//    (ORDER BY code DESC), so an untouched select saves '4'.
//  - FMS: radios labelled "<label>. <description>", stored as the fms_list id.

type Row = Record<string, string | number | null>;

interface Choice {
    value: string | number | null;
    label: string | null;
    description?: string | null;
}

interface Section {
    fields: Record<string, string | number | null>;
    lists: Record<string, Row[]>;
    version: string;
}

interface VisitResponses {
    visitId: number;
    visitDate: string | null;
    items: Section;
    medications: Section;
    details: Section;
}

// With ?language=… the backend returns {value, label} pairs (label = il8n_sp text, falling back to English)
// instead of bare strings; the stored value is always the English one, as in legacy.
type LabelledOption = string | { value: string; label: string | null };

interface NormalizedOptions {
    relationships: string[];
    walkingSupport: Choice[];
    walkingChanges: Choice[];
    fms: Choice[];
    devices: string[];
    gaitConcerns: Choice[];
    pain: string[];
}

interface HistoryOptions {
    relationships: LabelledOption[];
    walkingSupport: Choice[];
    walkingChanges: Choice[];
    fms: Choice[];
    devices: LabelledOption[];
    gaitConcerns: Choice[];
    pain: LabelledOption[];
}

export const REQUIRED_MESSAGE = 'Please fill in all answers before moving to the next question.';

export interface BaselineWho {
    person: string | null;
    relationship: string | null;
}

@Component({
    selector: 'app-history-visit-questionnaire',
    standalone: true,
    imports: [CommonModule, FormsModule],
    templateUrl: './history-visit-questionnaire.html',
    styleUrl: './history-visit-questionnaire.css'
})
export class HistoryVisitQuestionnaire implements OnChanges {
    @Input() patientId: number | null = null;
    @Input() visitId: number | null = null;
    @Input() firstName = '';
    @Input() hideActions = false;
    @Input() language: QuestionnaireLanguage = 'en';
    @Input() showWho = true;
    @Input() showGeneral = true;
    @Input() showGait = true;
    @Input() showConcerns = true;
    @Input() copyWho: BaselineWho | null = null;
    @Output() saveComplete = new EventEmitter<void>();
    @Output() saveFailed = new EventEmitter<string>();

    // frequencies (code -> label); the legacy select lists them ORDER BY code DESC, no blank option.
    // frequencies (ORDER BY code DESC, no blank option); replaced by /api/reference/frequencies once loaded.
    frequencyChoices: { value: string; label: string }[] = [
        { value: '4', label: 'None' },
        { value: '3', label: 'About once per month' },
        { value: '2', label: 'Two or three times per month' },
        { value: '1', label: 'One or more times per week' }
    ];

    readonly therapyGroups = [
        { title: 'PHYSICAL THERAPY', fields: [
            { key: 'hospitalPt', label: 'Hospital' }, { key: 'clinicPt', label: 'Clinic' },
            { key: 'schoolPt', label: 'School' }, { key: 'homePt', label: 'Home' }] },
        { title: 'OCCUPATIONAL THERAPY', fields: [
            { key: 'hospitalOt', label: 'Hospital' }, { key: 'clinicOt', label: 'Clinic' },
            { key: 'schoolOt', label: 'School' }, { key: 'homeOt', label: 'Home' }] }
    ];

    readonly deviceSideOptions = [
        { value: 'Both', label: 'Both' },
        { value: 'R', label: 'R only' },
        { value: 'L', label: 'L only' }
    ];

    // q-fms5/50/500.jsp question text.
    readonly fmsQuestions = [
        { key: 'fms5Id', text: 'How does [name] move around for short distances in the house?' },
        { key: 'fms50Id', text: 'How does [name] move around in and between classes at school?' },
        { key: 'fms500Id', text: 'How does [name] move around for long distances such as the shopping center?' }
    ];

    loading = false;
    loadError = '';
    saving = false;
    saveError = '';
    saved = false;

    items: Section | null = null;
    details: Section | null = null;
    medications: Section | null = null;
    options: NormalizedOptions | null = null;

    // Patient-level (pt_id-scoped) history.
    healthHistory: HealthHistoryEntry[] = [];
    healthConditions: HealthConditionEntry[] = [];
    botox: BotoxHistory[] = [];

    historyConditionOptions: HistoryConditionOption[] = [];
    ageOptions: string[] = [];
    conditionOptions: ConditionOption[] = [];
    bodyLocationOptions: string[] = [];
    botoxSideOptions: SideOption[] = [];
    seizureMedOptions: string[] = [];

    newHistoryConditionCode = '';
    newHistoryAge = '';
    historyError = '';
    historySaving = false;

    conditionError = '';
    confirmingConditionId: number | null = null;

    showCustomSeizureMed = false;
    customSeizureMed = '';

    showBotoxForm = false;
    newBotox = { bodyLocation: '', date: '', side: '', facility: '', physician: '' };
    botoxSaving = false;
    botoxError = '';

    showCustomDevice = false;
    customDevice = '';
    pendingDeviceSide: Record<string, string> = {};

    constructor(private http: HttpClient, private cdr: ChangeDetectorRef) {}

    // Only a change of what is being loaded resets the form — not e.g. the parent's live `copyWho`/`firstName`
    // updates, which would otherwise wipe whatever the user has already entered.
    ngOnChanges(changes: SimpleChanges): void {
        if (!(changes['patientId'] || changes['visitId'] || changes['language'] || changes['showWho'] || changes['showGeneral'] || changes['showGait'] || changes['showConcerns'])) return;
        this.saved = false;
        this.saveError = '';
        this.items = null;
        this.details = null;
        this.medications = null;
        this.options = null;
        this.healthHistory = [];
        this.healthConditions = [];
        this.botox = [];
        this.newHistoryConditionCode = '';
        this.newHistoryAge = '';
        this.historyError = '';
        this.conditionError = '';
        this.pendingConditions = new Set<string>();
        this.confirmingConditionId = null;
        this.pendingRemoval = null;
        this.showCustomSeizureMed = false;
        this.customSeizureMed = '';
        this.showBotoxForm = false;
        this.newBotox = { bodyLocation: '', date: '', side: '', facility: '', physician: '' };
        this.botoxError = '';
        this.showCustomDevice = false;
        this.customDevice = '';
        this.pendingDeviceSide = {};
        this.load();
    }

    get langParams(): { language: string } {
        return { language: this.language };
    }

    // Spanish (or English) display text per stored English value, filled from the {value,label} lists.
    private labels: Record<'relationship' | 'age' | 'device' | 'pain' | 'bodyLoc', Record<string, string>> =
        { relationship: {}, age: {}, device: {}, pain: {}, bodyLoc: {} };

    private normalize(kind: 'relationship' | 'age' | 'device' | 'pain' | 'bodyLoc', list: LabelledOption[] | undefined): string[] {
        return (list ?? []).map(item => {
            if (typeof item === 'string') return item;
            if (item.label) this.labels[kind][item.value] = item.label;
            return item.value;
        });
    }

    lab(kind: 'relationship' | 'age' | 'device' | 'pain' | 'bodyLoc', value: string | number | null | undefined): string {
        const key = String(value ?? '');
        return this.labels[kind][key] ?? key;
    }

    // Display names of stored condition rows, from the (localized) option lists, by code.
    conditionName(code: string | null | undefined, fallback: string | null | undefined): string {
        const found = this.conditionOptions.find(option => option.code === code);
        return found?.name ?? fallback ?? '';
    }

    historyConditionName(code: string | null | undefined, fallback: string | null | undefined): string {
        const found = this.historyConditionOptions.find(option => option.code === code);
        return found?.name ?? fallback ?? '';
    }

    // Walking-change labels are hard-coded in legacy (no DB table); the backend sends no Spanish for them.
    walkingChangeLabel(choice: Choice): string {
        const english: Record<string, string> = {
            'No Change': 'No Change', 'Much Better': 'Walks Much Better', 'Little Better': 'Walks a Little Better',
            'Little Worse': 'Walks a Little Worse', 'Much Worse': 'Walks Much Worse'
        };
        return choice.label && this.language === 'en' ? choice.label : this.text(english[String(choice.value)] ?? String(choice.label ?? choice.value ?? ''));
    }

    // Legacy wording, in the session language where the legacy page has a Spanish branch (legacy-i18n.ts),
    // with the patient's first name substituted for [name].
    text(key: string): string {
        return fillName(translate(key, this.language), this.firstName, this.language === 'sp' ? 'su niño(a)' : 'your child');
    }

    // Local (not yet saved) list rows go through legacy's confirmation page before being removed
    // (q-devices_update / q-seizures_update / q-concern_update).
    pendingRemoval: { kind: 'device' | 'seizure' | 'concern'; key: string | number | null; label: string } | null = null;

    askRemove(kind: 'device' | 'seizure' | 'concern', key: string | number | null, label: string): void {
        this.pendingRemoval = { kind, key, label };
    }

    cancelRemove(): void {
        this.pendingRemoval = null;
    }

    // q-concern_update.jsp has no Spanish for its heading / question / confirm button; only the cancel link.
    removalText(part: 'title' | 'question' | 'confirm' | 'cancel'): string {
        const kind = this.pendingRemoval?.kind;
        const englishOnly = kind === 'concern' && part !== 'cancel';
        const lang: QuestionnaireLanguage = englishOnly ? 'en' : this.language;
        const key = part === 'title'
            ? (kind === 'device' ? 'Confirm Device Deletion' : kind === 'seizure' ? 'Confirm Medication Deletion' : 'Confirm Gait Concern Deletion')
            : part === 'question' ? 'Are you sure you want to remove the following entry from your list?'
            : part === 'confirm' ? (kind === 'concern' ? 'Yes - Delete this from my List' : 'Yes - Delete this from my List#dot')
            : 'Do Not Delete this Item';
        return translate(key, lang);
    }

    confirmRemove(): void {
        const removal = this.pendingRemoval;
        this.pendingRemoval = null;
        if (!removal) return;
        if (removal.kind === 'device') this.removeDevice(String(removal.key));
        else if (removal.kind === 'seizure') this.removeSeizureMed(String(removal.key));
        else this.removeGaitConcern(removal.key);
    }

    private base(): string {
        return `/api/patients/${this.patientId}/questionnaire-responses`;
    }

    private patientBase(): string {
        return `/api/patients/${this.patientId}`;
    }

    private load(): void {
        if (!this.patientId || !this.visitId) {
            return;
        }

        this.loading = true;
        this.loadError = '';

        forkJoin({
            visit: this.http.get<VisitResponses>(`${this.base()}/visits/${this.visitId}`),
            options: this.http.get<HistoryOptions>(`${this.base()}/options`, { params: this.langParams }),
            patientHistory: this.http.get<PatientHistoryData>(`${this.patientBase()}/history`),
            historyConditions: this.http.get<HistoryConditionOption[]>('/api/reference/history-conditions', { params: this.langParams }),
            ages: this.http.get<LabelledOption[]>('/api/reference/ages', { params: this.langParams }),
            conditions: this.http.get<ConditionOption[]>('/api/reference/conditions', { params: this.langParams }),
            bodyLocations: this.http.get<LabelledOption[]>('/api/reference/body-locations', { params: this.langParams }),
            botoxSides: this.http.get<SideOption[]>('/api/reference/botox-sides', { params: this.langParams }),
            seizureMedOptions: this.http.get<string[]>('/api/reference/seizure-meds'),
            frequencies: this.http.get<{ value: string; label: string | null }[]>('/api/reference/frequencies', { params: this.langParams })
        }).subscribe({
            next: result => {
                this.items = result.visit.items;
                this.details = result.visit.details;
                this.medications = result.visit.medications;
                this.labels = { relationship: {}, age: {}, device: {}, pain: {}, bodyLoc: {} };
                this.options = {
                    ...result.options,
                    relationships: this.normalize('relationship', result.options.relationships),
                    devices: this.normalize('device', result.options.devices),
                    pain: this.normalize('pain', result.options.pain)
                };
                this.healthHistory = result.patientHistory.healthHistory ?? [];
                this.healthConditions = result.patientHistory.healthConditions ?? [];
                this.botox = result.patientHistory.botox ?? [];
                this.historyConditionOptions = result.historyConditions ?? [];
                this.ageOptions = this.normalize('age', result.ages);
                this.conditionOptions = result.conditions ?? [];
                this.bodyLocationOptions = this.normalize('bodyLoc', result.bodyLocations);
                this.botoxSideOptions = result.botoxSides ?? [];
                this.seizureMedOptions = result.seizureMedOptions ?? [];
                if (result.frequencies?.length) {
                    this.frequencyChoices = result.frequencies.map(item => ({ value: String(item.value), label: item.label ?? String(item.value) }));
                }
                this.applyLegacyDefaults();
                this.loading = false;
                this.cdr.markForCheck();
            },
            error: error => {
                console.error('Unable to load History:', error);
                this.loading = false;
                this.loadError = 'Unable to load this questionnaire.';
                this.cdr.markForCheck();
            }
        });
    }

    // q-pt.jsp's selects have no blank option, so an untouched select is submitted as its first
    // option, "None" (code 4).
    private applyLegacyDefaults(): void {
        if (!this.details) return;
        for (const group of this.therapyGroups) {
            for (const field of group.fields) {
                const current = this.details.fields[field.key];
                if (current === null || current === undefined || current === '') {
                    this.details.fields[field.key] = '4';
                }
            }
        }
    }

    // --- generic item/detail field helpers -----------------------------------------------------
    setItem(key: string, value: string | number | null): void {
        if (this.items) this.items.fields[key] = value;
    }

    // --- Health History (pt_health_history) ----------------------------------------------------
    // Legacy lists every history_cond problem (a condition can be added again with a different age)
    // and has no delete on this page.
    private reloadPatientHistory(): void {
        if (!this.patientId) return;
        this.http.get<PatientHistoryData>(`${this.patientBase()}/history`).subscribe({
            next: history => {
                this.healthHistory = history.healthHistory ?? [];
                this.healthConditions = history.healthConditions ?? [];
                this.botox = history.botox ?? [];
                this.cdr.markForCheck();
            },
            error: error => {
                console.error('Unable to reload patient history:', error);
                this.cdr.markForCheck();
            }
        });
    }

    addHealthHistory(): void {
        if (!this.patientId || !this.newHistoryConditionCode || !this.newHistoryAge) {
            // q-add_history.jsp check_ok(): the age select is checked first, then the condition radio.
            this.historyError = !this.newHistoryAge
                ? translate(REQUIRED_MESSAGE, this.language)
                : 'Please select a condition before moving to the next question.';
            return;
        }
        this.historySaving = true;
        this.historyError = '';
        this.http.post<HealthHistoryEntry>(`${this.patientBase()}/history/health-history`, {
            age: this.newHistoryAge,
            conditionCode: this.newHistoryConditionCode
        }).subscribe({
            next: () => {
                this.newHistoryConditionCode = '';
                this.newHistoryAge = '';
                this.historySaving = false;
                this.reloadPatientHistory();
            },
            error: error => {
                console.error('Unable to add health history item:', error);
                this.historySaving = false;
                this.historyError = 'Unable to add. Please try again.';
                this.cdr.markForCheck();
            }
        });
    }

    // --- Health Conditions (pt_health_conditions) ----------------------------------------------
    availableConditionOptions(): ConditionOption[] {
        const existing = new Set(this.healthConditions.map(item => (item.conditionCode ?? '').trim().toLowerCase()));
        return this.conditionOptions.filter(option => !existing.has(option.code.trim().toLowerCase()));
    }

    // q-conditions.jsp is ONE form: tick any number of boxes (un-ticking is free), then "Next Question"
    // adds them all. Ticks are held here and posted by save(); Delete stays immediate, as in legacy.
    pendingConditions = new Set<string>();

    isConditionPending(code: string): boolean {
        return this.pendingConditions.has(code);
    }

    togglePendingCondition(code: string): void {
        if (this.pendingConditions.has(code)) {
            this.pendingConditions.delete(code);
        } else {
            this.pendingConditions.add(code);
        }
    }

    // Legacy goes through a confirmation page (q-conditions_update.jsp) before deleting.
    askDeleteCondition(id: number): void {
        this.confirmingConditionId = id;
    }

    cancelDeleteCondition(): void {
        this.confirmingConditionId = null;
    }

    get confirmingCondition(): HealthConditionEntry | undefined {
        return this.healthConditions.find(entry => entry.id === this.confirmingConditionId);
    }

    confirmDeleteCondition(): void {
        const id = this.confirmingConditionId;
        if (!this.patientId || id === null) return;
        this.http.delete<void>(`${this.patientBase()}/history/health-conditions/${id}`).subscribe({
            next: () => {
                this.confirmingConditionId = null;
                this.reloadPatientHistory();
            },
            error: error => {
                console.error('Unable to remove health condition:', error);
                this.conditionError = 'Unable to remove. Please try again.';
                this.cdr.markForCheck();
            }
        });
    }

    // Legacy shows the seizure-medication page when a "Seizures" condition (code SEIZ) is recorded.
    // (A "Seizures" box ticked but not yet saved counts too, since everything is on one page here.)
    get hasSeizuresCondition(): boolean {
        const recorded = this.healthConditions.some(entry => {
            const text = `${entry.conditionCode ?? ''} ${entry.conditionDescription ?? ''}`.toLowerCase();
            return entry.conditionCode === 'SEIZ' || text.includes('seizure');
        });
        // Match by code, not the (possibly Spanish) name.
        return recorded || this.pendingConditions.has('SEIZ');
    }

    // --- Seizure medications (pt_seizure_meds, per visit) ----------------------------------------
    get seizureMedRows(): Row[] {
        return this.medications?.lists['medications'] ?? [];
    }

    rowName(row: Row): string {
        return String(row['name'] ?? '');
    }

    availableSeizureMeds(): string[] {
        const added = new Set(this.seizureMedRows.map(row => this.rowName(row).toLowerCase()));
        return this.seizureMedOptions.filter(name => !added.has(name.toLowerCase()));
    }

    addSeizureMed(name: string): void {
        const trimmed = name.trim();
        if (!trimmed || !this.medications) return;
        const rows = this.medications.lists['medications'] ?? [];
        this.medications.lists['medications'] = [...rows, { id: null, name: trimmed }];
    }

    addCustomSeizureMed(): void {
        this.addSeizureMed(this.customSeizureMed);
        this.customSeizureMed = '';
        this.showCustomSeizureMed = false;
    }

    removeSeizureMed(name: string): void {
        if (!this.medications) return;
        const rows = this.medications.lists['medications'] ?? [];
        this.medications.lists['medications'] = rows.filter(row => row['name'] !== name);
    }

    // --- Botox (pt_botox) ------------------------------------------------------------------------
    // The add page has no required fields and no delete; date is free text in legacy.
    addBotox(): void {
        if (!this.patientId) return;
        this.botoxSaving = true;
        this.botoxError = '';
        this.http.post<BotoxHistory>(`${this.patientBase()}/history/botox`, this.newBotox).subscribe({
            next: () => {
                this.newBotox = { bodyLocation: '', date: '', side: '', facility: '', physician: '' };
                this.botoxSaving = false;
                this.showBotoxForm = false;
                this.reloadPatientHistory();
            },
            error: error => {
                console.error('Unable to add botox shot:', error);
                this.botoxSaving = false;
                this.botoxError = 'Unable to add. Please try again.';
                this.cdr.markForCheck();
            }
        });
    }

    // --- Devices (pt_devices, per visit) ---------------------------------------------------------
    get deviceRows(): Row[] {
        return this.details?.lists['devices'] ?? [];
    }

    deviceHasSide(name: string): boolean {
        return name === 'Crutch' || name === 'Cane';
    }

    availableDevices(): string[] {
        const added = new Set(this.deviceRows.map(row => String(row['name'] ?? '')));
        return (this.options?.devices ?? []).filter(name => !added.has(name));
    }

    pendingSide(name: string): string {
        return this.pendingDeviceSide[name] ?? 'Both';
    }

    setPendingSide(name: string, side: string): void {
        this.pendingDeviceSide[name] = side;
    }

    addDevice(name: string): void {
        if (!this.details) return;
        const rows = this.details.lists['devices'] ?? [];
        this.details.lists['devices'] = [...rows, { id: null, name, side: this.deviceHasSide(name) ? this.pendingSide(name) : null }];
    }

    addCustomDevice(): void {
        const name = this.customDevice.trim();
        if (name && this.details) {
            const rows = this.details.lists['devices'] ?? [];
            this.details.lists['devices'] = [...rows, { id: null, name, side: null }];
        }
        this.customDevice = '';
        this.showCustomDevice = false;
    }

    removeDevice(name: string): void {
        if (!this.details) return;
        const rows = this.details.lists['devices'] ?? [];
        this.details.lists['devices'] = rows.filter(row => row['name'] !== name);
    }

    // --- Pain (pt_pain, per visit) ---------------------------------------------------------------
    private normalizeSide(value: string | number | null | undefined): string | null {
        const side = String(value ?? '');
        if (!side) return null;
        const lower = side.toLowerCase();
        return lower === 'left' ? 'L' : lower === 'right' ? 'R' : side;
    }

    painSideChoices(bodyPart: string): { value: string; label: string }[] {
        return bodyPart === 'Back'
            ? [{ value: 'Upper', label: this.text('Upper') }, { value: 'Lower', label: this.text('Lower') },
               { value: 'None', label: this.text('None') }]
            : [{ value: 'L', label: this.text('Left#pain') }, { value: 'R', label: this.text('Right#pain') },
               { value: 'Both', label: this.text('Both#pain') }, { value: 'None', label: this.text('None') }];
    }

    painSide(bodyPart: string): string {
        const row = this.details?.lists['pain']?.find(row => row['bodyPart'] === bodyPart);
        return row ? (this.normalizeSide(row['side']) ?? 'None') : 'None';
    }

    setPainSide(bodyPart: string, side: string): void {
        if (!this.details) return;
        const rows = this.details.lists['pain'] ?? [];
        const others = rows.filter(row => row['bodyPart'] !== bodyPart);
        if (side === 'None') {
            this.details.lists['pain'] = others;
            return;
        }
        const existing = rows.find(row => row['bodyPart'] === bodyPart);
        this.details.lists['pain'] = [...others, { id: existing ? existing['id'] : null, bodyPart, side }];
    }

    // --- Gait concerns (pt_gait_concerns, per visit) ---------------------------------------------
    get gaitConcernRows(): Row[] {
        return this.details?.lists['gaitConcerns'] ?? [];
    }

    gaitConcernLabel(code: string | number | null): string {
        const choice = (this.options?.gaitConcerns ?? []).find(item => item.value === code);
        return choice?.label ?? String(code ?? '');
    }

    availableGaitConcerns(): Choice[] {
        const added = new Set(this.gaitConcernRows.map(row => row['code']));
        return (this.options?.gaitConcerns ?? []).filter(choice => !added.has(choice.value));
    }

    addGaitConcern(code: string | number | null): void {
        if (!this.details) return;
        const rows = this.details.lists['gaitConcerns'] ?? [];
        this.details.lists['gaitConcerns'] = [...rows, { id: null, code }];
    }

    removeGaitConcern(code: string | number | null): void {
        if (!this.details) return;
        const rows = this.details.lists['gaitConcerns'] ?? [];
        this.details.lists['gaitConcerns'] = rows.filter(row => row['code'] !== code);
    }

    // --- validation ------------------------------------------------------------------------------
    // Mirrors the legacy client-side check_ok() of the pages that have one: q-who (name + relationship
    // must be filled), q-walking_support / q-walking_change / q-fms5/50/500 (one radio must be
    // checked). Every other visit-history page has no validation.
    validate(): string | null {
        const fields = this.items?.fields;
        if (!fields) return null;
        const blank = (value: unknown) => value === null || value === undefined || value === '' || value === '--';
        if (this.showWho && (blank(fields['reportingPerson']) || blank(fields['relationship']))) {
            return translate(REQUIRED_MESSAGE, this.language);
        }
        if (this.showGait) {
            const keys = ['walkingSupportCode', 'fms5Id', 'fms50Id', 'fms500Id', 'walkingChange'];
            if (keys.some(key => blank(fields[key]))) {
                return translate(REQUIRED_MESSAGE, this.language);
            }
        }
        return null;
    }

    // --- save ------------------------------------------------------------------------------------
    save(): void {
        if (!this.patientId || !this.visitId || !this.items || !this.details || !this.medications) {
            this.saveComplete.emit();
            return;
        }

        // First visit: nq-thanks.jsp copies the baseline person/relationship into the visit.
        if (!this.showWho && this.copyWho) {
            if (this.copyWho.person) this.items.fields['reportingPerson'] = this.copyWho.person;
            if (this.copyWho.relationship) this.items.fields['relationship'] = this.copyWho.relationship;
        }

        // Legacy's Dreamweaver update pages write a blank text field as '' (never NULL); do the same for the
        // text-typed columns. (FMS ids are numeric and stay null when unanswered.)
        for (const key of ['reportingPerson', 'relationship', 'walkingSupportCode', 'walkingChange']) {
            if (this.items.fields[key] === null || this.items.fields[key] === undefined) this.items.fields[key] = '';
        }
        for (const key of ['followup', 'generalConcerns']) {
            if (this.details.fields[key] === null || this.details.fields[key] === undefined) this.details.fields[key] = '';
        }

        this.saving = true;
        this.saveError = '';

        // 1) Add the ticked Health Conditions (patient-level rows), one at a time; each tick is dropped from the
        //    pending set as soon as its own POST succeeds, so a retry after a partial failure never re-adds it.
        // 2) Then the three visit-level PUTs, each on its own so one failing does not cancel (and leave
        //    half-sent) the others; they replace their scope wholesale, so re-sending all three on retry is safe.
        const pending = [...this.pendingConditions];
        const addConditions = from(pending).pipe(
            concatMap(code => this.http
                .post<HealthConditionEntry>(`${this.patientBase()}/history/health-conditions`, { conditionCode: code })
                .pipe(tap(() => this.pendingConditions.delete(code)))),
            toArray()
        );
        const put = (part: string, body: Section) => this.http
            .put<void>(`${this.base()}/visits/${this.visitId}/${part}`, body)
            .pipe(map(() => null as string | null), catchError(error => {
                console.error(`Unable to save History (${part}):`, error);
                return of(part as string | null);
            }));

        addConditions.pipe(
            switchMap(() => forkJoin([put('items', this.items!), put('details', this.details!), put('medications', this.medications!)]))
        ).subscribe({
            next: results => {
                if (pending.length > 0) this.reloadPatientHistory();
                const failed = results.filter((part): part is string => part !== null);
                this.saving = false;
                if (failed.length > 0) {
                    this.saveError = `Unable to save (${failed.join(', ')}). Please try again.`;
                    this.cdr.markForCheck();
                    this.saveFailed.emit(this.saveError);
                    return;
                }
                this.saved = true;
                this.cdr.markForCheck();
                this.saveComplete.emit();
            },
            error: error => {
                console.error('Unable to add health conditions:', error);
                if (pending.length !== this.pendingConditions.size) this.reloadPatientHistory();
                this.saving = false;
                this.saveError = 'Unable to save the ticked conditions. Please try again.';
                this.cdr.markForCheck();
                this.saveFailed.emit(this.saveError);
            }
        });
    }
}












<div class="history-form">
    @if (!visitId) {
    <div class="questionnaire-error">
        No specific visit is selected yet — pick a scheduled visit before filling this out.
    </div>
    } @else if (loading) {
    <div class="loading-text">Loading…</div>
    } @else if (loadError) {
    <div class="questionnaire-error">{{ loadError }}</div>
    } @else if (items && details && medications && options) {

    @if (saved) {
    <div class="submitted-banner">History saved.</div>
    }
    @if (saveError) {
    <div class="questionnaire-error">{{ saveError }}</div>
    }

    <!-- ============================ visit history pages ============================ -->
    @if (showGeneral) {

    @if (showWho) {
    <section class="history-page intro-page">
        <h3 class="page-heading">{{ text('Welcome to the duPont Hospital for Children Gait Lab') }}</h3>
        <p class="page-text">{{ text('In order to best understand our patients, we need your help with some history about [name].') }}</p>
        <p class="page-text">{{ text("If you have any questions while completing the questionnaire, please don't hesitate to ask a member of our staff for help.") }}</p>
        <p class="page-text">{{ text('These questions will be asked on each visit so we can stay up-to-date with you.') }}</p>
    </section>

    <section class="history-page">
        <h3 class="page-heading">{{ text('Please tell us about you') }}</h3>
        <div class="field-row">
            <label class="field-label" for="reportingPerson">{{ text('Your Name:') }}</label>
            <input id="reportingPerson" type="text" [ngModel]="items.fields['reportingPerson']"
                (ngModelChange)="setItem('reportingPerson', $event)" />
        </div>
        <div class="field-row">
            <label class="field-label" for="relationship">{{ text('What is your relationship to [name]?') }}
                <span class="field-hint">{{ text('Choose from the list') }}</span></label>
            <select id="relationship" [ngModel]="items.fields['relationship']"
                (ngModelChange)="setItem('relationship', $event)">
                <option [ngValue]="null"></option>
                @for (choice of options.relationships; track choice) {
                <option [ngValue]="choice">{{ lab('relationship', choice) }}</option>
                }
            </select>
        </div>
    </section>
    }

    <!-- Health History -->
    <section class="history-page">
        <h3 class="page-heading">{{ text('Health History') }}</h3>
        @if (healthHistory.length > 0) {
        <p class="page-text">{{ text("Here is what we show for [name]'s Health History:") }}</p>
        <table class="entry-table">
            <thead><tr><th>{{ text('Problem') }}</th><th>{{ text('Age') }}</th></tr></thead>
            <tbody>
                @for (entry of healthHistory; track entry.id) {
                <tr><td>{{ historyConditionName(entry.conditionCode, entry.conditionDescription) }}</td><td>{{ lab('age', entry.age) }}</td></tr>
                }
            </tbody>
        </table>
        }
        <p class="page-text">
            {{ healthHistory.length > 0
                ? text('Has [name] had any of the following problems that are not included in the chart above?')
                : text('Has [name] had any of the following problems?') }}
        </p>
        <h4 class="sub-heading">{{ text('Add to the Health History') }}</h4>
        <p class="page-text">{{ text('Please select the problem and age that [name] was when it happened. If you are not sure, select "Not Sure" in the Age list.') }}</p>
        <div class="add-history">
            <div class="radio-column">
                <span class="field-label">{{ text('Select One Problem') }}</span>
                @for (option of historyConditionOptions; track option.code) {
                <label class="choice-row">
                    <input type="radio" name="historyCondition" [value]="option.code"
                        [checked]="newHistoryConditionCode === option.code"
                        (change)="newHistoryConditionCode = option.code" />
                    {{ option.name }}
                </label>
                }
            </div>
            <div class="age-column">
                <span class="field-label">{{ text('Age') }}</span>
                <select [(ngModel)]="newHistoryAge">
                    <option value=""></option>
                    @for (age of ageOptions; track age) {
                    <option [value]="age">{{ lab('age', age) }}</option>
                    }
                </select>
                <button type="button" class="add-btn" [disabled]="historySaving" (click)="addHealthHistory()">
                    {{ text('Add This To Record') }}
                </button>
            </div>
        </div>
        @if (historyError) {
        <div class="questionnaire-error">{{ historyError }}</div>
        }
    </section>

    <!-- Health Conditions -->
    <section class="history-page">
        <h3 class="page-heading">{{ text('Health Conditions') }}</h3>
        @if (healthConditions.length > 0) {
        <p class="page-text">
            {{ text('Here is the information in [name]\'s record to date. If the list is ok, move on to the next question. If you need to delete one of them from the list, click "Delete" in that row.') }}
        </p>
        <table class="entry-table">
            <thead><tr><th>{{ text('Condition') }}</th><th></th></tr></thead>
            <tbody>
                @for (entry of healthConditions; track entry.id) {
                <tr>
                    <td>{{ conditionName(entry.conditionCode, entry.conditionDescription) }}</td>
                    <td class="row-action"><button type="button" class="link-btn"
                            (click)="askDeleteCondition(entry.id)">{{ text('Delete') }}</button></td>
                </tr>
                }
            </tbody>
        </table>
        }
        @if (confirmingCondition; as pending) {
        <div class="confirm-box">
            <h4 class="sub-heading">{{ text('Confirm Condition Deletion') }}</h4>
            <p class="page-text"><strong>{{ text('Are you sure you want to remove the following entry from your list?') }}</strong></p>
            <p class="page-text">{{ conditionName(pending.conditionCode, pending.conditionDescription) }}</p>
            <div class="confirm-actions">
                <button type="button" class="add-btn" (click)="confirmDeleteCondition()">{{ text('Yes - Delete this from my List') }}</button>
                <button type="button" class="link-btn" (click)="cancelDeleteCondition()">{{ text('Do Not Delete this Item') }}</button>
            </div>
        </div>
        }
        <p class="page-text">
            {{ healthConditions.length > 0
                ? text('Does [name] have any other conditions that are listed below?')
                : text('Does [name] have any conditions that are listed below?') }}
            {{ text('If so, please select them then move to the next question.') }}
        </p>
        <div class="check-grid">
            @for (option of availableConditionOptions(); track option.code) {
            <label class="choice-row">
                <input type="checkbox" [checked]="isConditionPending(option.code)" (change)="togglePendingCondition(option.code)" />
                {{ option.name }}
            </label>
            }
        </div>
        @if (conditionError) {
        <div class="questionnaire-error">{{ conditionError }}</div>
        }
    </section>

    <!-- Seizure Medications: only when Seizures is among the recorded conditions -->
    @if (hasSeizuresCondition) {
    <section class="history-page">
        <h3 class="page-heading">{{ text('Seizure Medications') }}</h3>
        @if (seizureMedRows.length > 0) {
        <p class="page-text">{{ text('Here is the information you\'ve given so far today. If the list is ok, move on to the next question. If you need to delete one of them from the list, click "Delete" in that row.') }}</p>
        <table class="entry-table">
            <thead><tr><th>{{ text('Medication Name') }}</th><th></th></tr></thead>
            <tbody>
                @for (row of seizureMedRows; track row['name']) {
                <tr>
                    <td>{{ row['name'] }}</td>
                    <td class="row-action"><button type="button" class="link-btn"
                            (click)="askRemove('seizure', rowName(row), rowName(row))">{{ text('Delete') }}</button></td>
                </tr>
                }
            </tbody>
        </table>
        }
        @if (pendingRemoval && pendingRemoval.kind === 'seizure') {
        <div class="confirm-box">
            <h4 class="sub-heading">{{ removalText('title') }}</h4>
            <p class="page-text"><strong>{{ removalText('question') }}</strong></p>
            <p class="page-text">{{ pendingRemoval.label }}</p>
            <div class="confirm-actions">
                <button type="button" class="add-btn" (click)="confirmRemove()">{{ removalText('confirm') }}</button>
                <button type="button" class="link-btn" (click)="cancelRemove()">{{ removalText('cancel') }}</button>
            </div>
        </div>
        }
        @if (!showCustomSeizureMed) {
        <p class="page-text">
            {{ seizureMedRows.length > 0
                ? text('Does [name] currently use any other seizure medications? If so, please select them then move to the next question.')
                : text('Does [name] currently use any seizure medications? If so, please select them then move to the next question.') }}
        </p>
        <div class="check-grid">
            @for (med of availableSeizureMeds(); track med) {
            <label class="choice-row">
                <input type="checkbox" [checked]="false" (change)="addSeizureMed(med)" />
                {{ med }}
            </label>
            }
        </div>
        <button type="button" class="link-btn" (click)="showCustomSeizureMed = true">
            {{ text('Click here to enter a medication that is not in the list') }}
        </button>
        } @else {
        <div class="field-row">
            <label class="field-label" for="customSeizureMed">{{ text('Enter name of Medication:') }}</label>
            <input id="customSeizureMed" type="text" [(ngModel)]="customSeizureMed"
                (keydown.enter)="addCustomSeizureMed()" />
        </div>
        <div class="confirm-actions">
            <button type="button" class="add-btn" (click)="addCustomSeizureMed()">{{ text('Add this Medication') }}</button>
            <button type="button" class="link-btn" (click)="showCustomSeizureMed = false">
                {{ text('Click here to return to the previous page without adding a new medication') }}
            </button>
        </div>
        }
    </section>
    }

    <!-- Botox -->
    <section class="history-page">
        <h3 class="page-heading">{{ text('Botox Shot Information') }}</h3>
        <p class="page-text"><em>{{ text('A botox shot is a shot placed into tight muscles to loosen them.') }}</em></p>
        @if (!showBotoxForm) {
        <p class="page-text">
            {{ botox.length > 0 ? text('Below is the information we have on file regarding past Botox injections.') : text('No Shots on Record') }}
        </p>
        @if (botox.length > 0) {
        <table class="entry-table">
            <thead><tr><th>{{ text('Where') }}</th><th>{{ text('Date') }}</th><th>{{ text('Side') }}</th><th>{{ text('Hospital/Clinic') }}</th><th>{{ text('Physician') }}</th></tr></thead>
            <tbody>
                @for (entry of botox; track entry.id) {
                <tr>
                    <td>{{ lab('bodyLoc', entry.bodyLocation) }}</td><td>{{ entry.date }}</td><td>{{ entry.side }}</td>
                    <td>{{ entry.facility }}</td><td>{{ entry.physician }}</td>
                </tr>
                }
            </tbody>
        </table>
        }
        <p class="page-text">{{ text('Please select an option:') }}</p>
        <button type="button" class="link-btn" (click)="showBotoxForm = true">{{ text('Add a New Shot') }}</button>
        } @else {
        <h4 class="sub-heading">{{ text('Botox Shot Information - Add a New Shot') }}</h4>
        <div class="botox-form">
            <div class="field-row">
                <label class="field-label" for="botoxBodyLocation">{{ text('Body Location') }}</label>
                <select id="botoxBodyLocation" [(ngModel)]="newBotox.bodyLocation">
                    <option value=""></option>
                    @for (option of bodyLocationOptions; track option) {
                    <option [value]="option">{{ lab('bodyLoc', option) }}</option>
                    }
                </select>
            </div>
            <div class="field-row">
                <label class="field-label" for="botoxDate">{{ text('Date') }}</label>
                <input id="botoxDate" type="text" [(ngModel)]="newBotox.date" />
            </div>
            <div class="field-row">
                <span class="field-label">{{ text('Side') }}</span>
                <div class="inline-choices">
                    @for (option of botoxSideOptions; track option.value) {
                    <label class="choice-row">
                        <input type="radio" name="botoxSide" [value]="option.value"
                            [checked]="newBotox.side === option.value" (change)="newBotox.side = option.value" />
                        {{ text(option.label) }}
                    </label>
                    }
                </div>
            </div>
            <div class="field-row">
                <label class="field-label" for="botoxFacility">{{ text('Hospital/Clinic#form') }}</label>
                <input id="botoxFacility" type="text" [(ngModel)]="newBotox.facility" />
            </div>
            <div class="field-row">
                <label class="field-label" for="botoxPhysician">{{ text('Physician') }}</label>
                <input id="botoxPhysician" type="text" [(ngModel)]="newBotox.physician" />
            </div>
        </div>
        <div class="confirm-actions">
            <button type="button" class="add-btn" [disabled]="botoxSaving" (click)="addBotox()">{{ text('Add This To Record') }}</button>
            <button type="button" class="link-btn" (click)="showBotoxForm = false">{{ text('Cancel Add Shot') }}</button>
        </div>
        @if (botoxError) {
        <div class="questionnaire-error">{{ botoxError }}</div>
        }
        }
    </section>

    <!-- Devices -->
    <section class="history-page">
        <h3 class="page-heading">{{ text('Devices and Braces') }}</h3>
        @if (deviceRows.length > 0) {
        <p class="page-text">{{ text('Here is the information you\'ve given so far today. If the list is ok, move on to the next question. If you need to delete one of them from the list, click "Delete" in that row.') }}</p>
        <table class="entry-table">
            <thead><tr><th>{{ text('Device') }}</th><th>{{ text('Side') }}</th><th></th></tr></thead>
            <tbody>
                @for (row of deviceRows; track row['name']) {
                <tr>
                    <td>{{ lab('device', row['name']) }}</td><td>{{ row['side'] }}</td>
                    <td class="row-action"><button type="button" class="link-btn"
                            (click)="askRemove('device', rowName(row), lab('device', rowName(row)))">{{ text('Delete') }}</button></td>
                </tr>
                }
            </tbody>
        </table>
        }
        @if (pendingRemoval && pendingRemoval.kind === 'device') {
        <div class="confirm-box">
            <h4 class="sub-heading">{{ removalText('title') }}</h4>
            <p class="page-text"><strong>{{ removalText('question') }}</strong></p>
            <p class="page-text">{{ pendingRemoval.label }}</p>
            <div class="confirm-actions">
                <button type="button" class="add-btn" (click)="confirmRemove()">{{ removalText('confirm') }}</button>
                <button type="button" class="link-btn" (click)="cancelRemove()">{{ removalText('cancel') }}</button>
            </div>
        </div>
        }
        @if (!showCustomDevice) {
        <p class="page-text">
            {{ deviceRows.length > 0
                ? text('Does [name] currently use any other devices? If so, please select them then move to the next question.')
                : text('Does [name] currently use any devices? If so, please select them then move to the next question.') }}
        </p>
        <div class="check-grid">
            @for (device of availableDevices(); track device) {
            <div class="choice-row device-choice">
                <label class="device-label">
                    <input type="checkbox" [checked]="false" (change)="addDevice(device)" />
                    {{ lab('device', device) }}
                </label>
                @if (deviceHasSide(device)) {
                <span class="dash">--</span>
                <select class="side-select" [ngModel]="pendingSide(device)"
                    (ngModelChange)="setPendingSide(device, $event)">
                    @for (side of deviceSideOptions; track side.value) {
                    <option [ngValue]="side.value">{{ text(side.label === 'Both' ? 'Both#side' : side.label) }}</option>
                    }
                </select>
                }
            </div>
            }
        </div>
        <button type="button" class="link-btn" (click)="showCustomDevice = true">
            {{ text('Click here to enter a device that is not in the list') }}
        </button>
        } @else {
        <h4 class="sub-heading">{{ text('Device Entry') }}</h4>
        <div class="field-row">
            <label class="field-label" for="customDevice">{{ text('Enter name of Device:') }}</label>
            <input id="customDevice" type="text" [(ngModel)]="customDevice" (keydown.enter)="addCustomDevice()" />
        </div>
        <div class="confirm-actions">
            <button type="button" class="add-btn" (click)="addCustomDevice()">{{ text('Add this Device') }}</button>
            <button type="button" class="link-btn" (click)="showCustomDevice = false">
                {{ text('Click here to return to the previous page without adding a new device') }}
            </button>
        </div>
        }
    </section>

    <!-- Pain -->
    <section class="history-page">
        <h3 class="page-heading">{{ text('Pain Information') }}</h3>
        <p class="page-text">
            {{ (details.lists['pain'] ?? []).length > 0
                ? text('Does [name] have pain in any other places listed? If so, please select where the pain is located then move to the next question.')
                : text('Does [name] have pain in any places listed? If so, please select where the pain is located then move to the next question.') }}
        </p>
        <div class="pain-list">
            @for (bodyPart of options.pain; track bodyPart) {
            <div class="pain-row">
                <span class="pain-part">{{ lab('pain', bodyPart) }}</span>
                <div class="pain-sides">
                    @for (choice of painSideChoices(bodyPart); track choice.value) {
                    <label class="pain-side-option">
                        <input type="radio" [name]="'pain-' + bodyPart" [value]="choice.value"
                            [checked]="painSide(bodyPart) === choice.value"
                            (change)="setPainSide(bodyPart, choice.value)" />
                        {{ choice.label }}
                    </label>
                    }
                </div>
            </div>
            }
        </div>
    </section>

    <!-- Physical Therapy -->
    <section class="history-page">
        <h3 class="page-heading">{{ text('Physical Therapy Services') }}</h3>
        <p class="page-text">
            {{ text('If [name] receives any therapy, please select the frequency of PT and/or OT below:') }}
        </p>
        @for (group of therapyGroups; track group.title) {
        <h4 class="sub-heading">{{ text(group.title) }}</h4>
        @for (field of group.fields; track field.key) {
        <div class="field-row inline">
            <label class="field-label" [for]="field.key">{{ text(field.label) }}</label>
            <select [id]="field.key" [(ngModel)]="details.fields[field.key]">
                @for (choice of frequencyChoices; track choice.value) {
                <option [ngValue]="choice.value">{{ choice.label }}</option>
                }
            </select>
        </div>
        }
        }
    </section>

    <!-- Followup -->
    <section class="history-page">
        <h3 class="page-heading">{{ text('Followup Date') }}</h3>
        <p class="page-text">{{ text('If you know the date of the followup appointment with your doctor, please enter it below:') }}</p>
        <input id="followup" type="text" class="followup-input" [(ngModel)]="details.fields['followup']" />
        <p class="page-text"><em>{{ text('(for example, 2/5/2004 or Sept 2003)') }}</em></p>
    </section>

    }

    <!-- ============================ gait history pages ============================ -->
    @if (showGait) {

    <section class="history-page">
        <h3 class="page-heading">{{ text('Walking Support') }}</h3>
        <p class="page-text">{{ text("Which currently describes [name]'s need for support while walking?") }}</p>
        <div class="radio-column">
            @for (choice of options.walkingSupport; track choice.value) {
            <label class="choice-row">
                <input type="radio" name="walkingSupportCode" [value]="choice.value"
                    [checked]="items.fields['walkingSupportCode'] === choice.value"
                    (change)="setItem('walkingSupportCode', choice.value)" />
                {{ choice.label }}
            </label>
            }
        </div>
    </section>

    @for (question of fmsQuestions; track question.key) {
    <section class="history-page">
        <h3 class="page-heading">{{ text(question.text) }}</h3>
        <div class="radio-column">
            @for (choice of options.fms; track choice.value) {
            <label class="choice-row">
                <input type="radio" [name]="question.key" [value]="choice.value"
                    [checked]="items.fields[question.key] === choice.value"
                    (change)="setItem(question.key, choice.value)" />
                {{ choice.label }}. {{ choice.description }}
            </label>
            }
        </div>
    </section>
    }

    <section class="history-page">
        <h3 class="page-heading">{{ text('Changes in Walking') }}</h3>
        <p class="page-text">{{ text("In the last 6-12 months (or since last visit), how has [name]'s ability to walk changed?") }}</p>
        <div class="radio-column">
            @for (choice of options.walkingChanges; track choice.value) {
            <label class="choice-row">
                <input type="radio" name="walkingChange" [value]="choice.value"
                    [checked]="items.fields['walkingChange'] === choice.value"
                    (change)="setItem('walkingChange', choice.value)" />
                {{ walkingChangeLabel(choice) }}
            </label>
            }
        </div>
    </section>

    <section class="history-page">
        <h3 class="page-heading">{{ text('Current Gait Concerns') }}</h3>
        @if (gaitConcernRows.length > 0) {
        <p class="page-text">{{ text('Here is the information you\'ve given so far today. If the list is ok, move on to the next question. If you need to delete one of them from the list, click "Delete" in that row.') }}</p>
        <table class="entry-table">
            <thead><tr><th>{{ text('Gait Concern') }}</th><th></th></tr></thead>
            <tbody>
                @for (row of gaitConcernRows; track row['code']) {
                <tr>
                    <td>{{ gaitConcernLabel(row['code']) }}</td>
                    <td class="row-action"><button type="button" class="link-btn"
                            (click)="askRemove('concern', row['code'], gaitConcernLabel(row['code']))">{{ text('Delete') }}</button></td>
                </tr>
                }
            </tbody>
        </table>
        }
        @if (pendingRemoval && pendingRemoval.kind === 'concern') {
        <div class="confirm-box">
            <h4 class="sub-heading">{{ removalText('title') }}</h4>
            <p class="page-text"><strong>{{ removalText('question') }}</strong></p>
            <p class="page-text">{{ pendingRemoval.label }}</p>
            <div class="confirm-actions">
                <button type="button" class="add-btn" (click)="confirmRemove()">{{ removalText('confirm') }}</button>
                <button type="button" class="link-btn" (click)="cancelRemove()">{{ removalText('cancel') }}</button>
            </div>
        </div>
        }
        <p class="page-text">
            {{ gaitConcernRows.length > 0
                ? text('Do you currently have any other concerns listed below? If so, please select them then move to the next question.')
                : text('Do you currently have any concerns listed below? If so, please select them then move to the next question.') }}
        </p>
        <div class="check-grid">
            @for (choice of availableGaitConcerns(); track choice.value) {
            <label class="choice-row">
                <input type="checkbox" [checked]="false" (change)="addGaitConcern(choice.value)" />
                {{ choice.label }}
            </label>
            }
        </div>
    </section>

    }

    <!-- ============================ concerns page ============================ -->
    @if (showConcerns) {
    <section class="history-page">
        @if (text('Any Concerns?')) {
        <h3 class="page-heading">{{ text('Any Concerns?') }}</h3>
        }
        <p class="page-text">{{ text('Please type in any other concerns you have today:') }}</p>
        <textarea id="generalConcerns" rows="4" [(ngModel)]="details.fields['generalConcerns']"></textarea>
    </section>
    }

    @if (!hideActions) {
    <div class="history-actions">
        <button type="button" class="submit-btn" [disabled]="saving" (click)="save()">
            {{ saving ? 'Saving...' : (saved ? 'Save Again' : 'Save') }}
        </button>
    </div>
    }

    }
</div>
















:host {
    display: block;
}

.history-form {
    display: flex;
    flex-direction: column;
    gap: 14px;
}

.loading-text {
    color: #6b7280;
    font-size: 14px;
}

.submitted-banner {
    padding: 12px 16px;
    background: #eef6f4;
    border: 1px solid #a6d8cf;
    border-radius: 8px;
    color: #1f6b5e;
    font-size: 13px;
    font-weight: 500;
}

.questionnaire-error {
    padding: 12px 14px;
    border: 1px solid #efc5c5;
    border-radius: 8px;
    background: #fff4f4;
    color: #b42318;
    font-size: 13px;
}

.history-body {
    display: flex;
    flex-direction: column;
    gap: 14px;
}

.subsection-title {
    margin: 10px 0 0;
    padding-left: 10px;
    border-left: 3px solid #94a8a6;
    color: #475569;
    font-size: 13px;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.04em;
}

.subsection-title:first-child {
    margin-top: 0;
}

.question-block {
    display: flex;
    flex-direction: column;
    gap: 12px;
    padding: 18px 20px;
    background: #ffffff;
    border: 1px solid #e2e8f0;
    border-radius: 10px;
    transition: border-color 0.15s ease, box-shadow 0.15s ease;
}

.question-block:hover {
    border-color: #cbd5e1;
    box-shadow: 0 1px 6px rgba(15, 23, 42, 0.05);
}

.question-block:focus-within {
    border-color: #269c96;
    box-shadow: 0 0 0 3px rgba(38, 156, 150, 0.12);
}

.question-label {
    font-size: 14px;
    font-weight: 600;
    color: #1e293b;
}

.question-block input[type="text"],
.question-block input[type="date"],
.question-block textarea,
.question-block select {
    width: 100%;
    max-width: 360px;
    padding: 10px 14px;
    box-sizing: border-box;
    border: 1px solid #d7dde2;
    border-radius: 8px;
    background: #ffffff;
    color: #263238;
    font-family: inherit;
    font-size: 13px;
}

.question-block textarea {
    max-width: none;
    resize: vertical;
}

.question-block input:focus,
.question-block textarea:focus,
.question-block select:focus {
    outline: none;
    border-color: #269c96;
    box-shadow: 0 0 0 3px rgba(38, 156, 150, 0.12);
}

.therapy-grid {
    display: flex;
    flex-direction: column;
    gap: 10px;
}

.therapy-item {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 14px;
    padding: 10px 14px;
    background: #f8fafc;
    border: 1px solid #eef1f4;
    border-radius: 8px;
}

.therapy-item label {
    font-size: 13px;
    color: #374151;
    font-weight: 500;
}

.therapy-item select {
    max-width: 240px;
    margin: 0;
}

.checklist {
    display: grid;
    grid-template-columns: repeat(2, minmax(0, 1fr));
    gap: 4px 16px;
}

.checklist-row {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
    padding: 9px 12px;
    border-radius: 8px;
    transition: background-color 0.12s ease;
}

.checklist-row:has(.checkbox-option input:checked) {
    background: #eef6f4;
}

.checkbox-option {
    display: flex;
    align-items: center;
    gap: 10px;
    font-size: 13px;
    color: #374151;
    cursor: pointer;
}

.checklist > .checkbox-option {
    padding: 9px 12px;
    border-radius: 8px;
    transition: background-color 0.12s ease;
}

.checklist > .checkbox-option:hover {
    background: #f8fafc;
}

.checklist > .checkbox-option:has(input:checked) {
    background: #eef6f4;
    color: #1f6b5e;
    font-weight: 500;
}

.checkbox-option input[type="checkbox"] {
    width: 16px;
    height: 16px;
    accent-color: #269c96;
    cursor: pointer;
    flex-shrink: 0;
}

.pain-list {
    display: flex;
    flex-direction: column;
    gap: 4px;
}

.pain-row {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
    padding: 8px 12px;
    border-radius: 8px;
}

.pain-row:hover {
    background: #f8fafc;
}

.pain-part {
    font-size: 13px;
    font-weight: 500;
    color: #374151;
}

.pain-sides {
    display: flex;
    gap: 16px;
}

.pain-side-option {
    display: flex;
    align-items: center;
    gap: 6px;
    font-size: 13px;
    color: #374151;
    cursor: pointer;
}

.pain-side-option input {
    width: 15px;
    height: 15px;
    accent-color: #269c96;
    cursor: pointer;
}

.side-select {
    max-width: 160px;
    margin: 0;
}

.quick-pick-list {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    margin-bottom: 4px;
}

.quick-pick-btn {
    padding: 6px 12px;
    background: #ffffff;
    border: 1px solid #d7dde2;
    border-radius: 999px;
    color: #374151;
    font-family: inherit;
    font-size: 12px;
    cursor: pointer;
    transition: background-color 0.12s ease, border-color 0.12s ease;
}

.quick-pick-btn:hover:not(:disabled) {
    background: #eef6f4;
    border-color: #269c96;
}

.quick-pick-btn:disabled {
    background: #f0f4f5;
    border-color: #e2e8f0;
    color: #9ca3af;
    cursor: not-allowed;
}

.add-row {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
}

.add-row input,
.add-row select {
    max-width: 220px;
    margin: 0;
}

.botox-add-row input,
.botox-add-row select {
    max-width: 170px;
}

.add-btn {
    height: 40px;
    padding: 0 16px;
    color: #269c96;
    background: #ffffff;
    border: 1px solid #269c96;
    border-radius: 8px;
    font-family: inherit;
    font-size: 13px;
    font-weight: 600;
    cursor: pointer;
    transition: background-color 0.12s ease;
}

.add-btn:hover {
    background: #eef6f4;
}

.chip-list {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    padding: 0;
    margin: 4px 0 0;
    list-style: none;
}

.chip {
    display: flex;
    align-items: center;
    gap: 6px;
    padding: 6px 8px 6px 14px;
    background: #f0f4f5;
    border: 1px solid #d7dde2;
    border-radius: 999px;
    font-size: 13px;
    color: #374151;
}

.chip-remove {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 20px;
    height: 20px;
    border: none;
    border-radius: 50%;
    background: none;
    color: #6b7280;
    font-size: 15px;
    line-height: 1;
    cursor: pointer;
    transition: background-color 0.12s ease, color 0.12s ease;
}

.chip-remove:hover {
    background: #fde8e8;
    color: #b42318;
}

.history-actions {
    display: flex;
    justify-content: flex-end;
    padding-top: 4px;
}

.submit-btn {
    height: 40px;
    padding: 0 22px;
    color: #ffffff;
    background: #269c96;
    border: 1px solid #1f847f;
    border-radius: 8px;
    font-family: inherit;
    font-size: 14px;
    font-weight: 600;
    cursor: pointer;
    box-shadow: 0 1px 3px rgba(31, 132, 127, 0.25);
    transition: background-color 0.15s ease, box-shadow 0.15s ease;
}

.submit-btn:hover:not(:disabled) {
    background: #1f847f;
    box-shadow: 0 2px 6px rgba(31, 132, 127, 0.3);
}

.submit-btn:disabled {
    background: #b7d4d2;
    border-color: #b7d4d2;
    box-shadow: none;
    cursor: not-allowed;
}

/* ---- legacy page structure ---- */
.history-page {
    display: flex;
    flex-direction: column;
    gap: 12px;
    padding: 18px 20px;
    background: #ffffff;
    border: 1px solid #e2e8f0;
    border-radius: 10px;
}

.intro-page {
    background: #f8fafc;
}

.page-heading {
    margin: 0;
    color: #1e293b;
    font-size: 16px;
    font-weight: 700;
}

.sub-heading {
    margin: 4px 0 0;
    color: #334155;
    font-size: 13px;
    font-weight: 700;
}

.page-text {
    margin: 0;
    color: #374151;
    font-size: 13px;
    line-height: 1.5;
}

.field-row {
    display: flex;
    flex-direction: column;
    gap: 6px;
}

.field-row.inline {
    flex-direction: row;
    align-items: center;
    justify-content: space-between;
    max-width: 520px;
}

.field-label {
    color: #1e293b;
    font-size: 13px;
    font-weight: 600;
}

.field-hint {
    margin-left: 8px;
    color: #6b7280;
    font-size: 12px;
    font-weight: 400;
}

.history-page input[type="text"],
.history-page input[type="date"],
.history-page select,
.history-page textarea {
    max-width: 360px;
    padding: 9px 12px;
    box-sizing: border-box;
    border: 1px solid #d7dde2;
    border-radius: 8px;
    background: #ffffff;
    color: #263238;
    font-family: inherit;
    font-size: 13px;
}

.history-page textarea {
    max-width: none;
    width: 100%;
    resize: vertical;
}

.history-page input:focus,
.history-page select:focus,
.history-page textarea:focus {
    outline: none;
    border-color: #269c96;
    box-shadow: 0 0 0 3px rgba(38, 156, 150, 0.12);
}

.entry-table {
    width: 100%;
    border-collapse: collapse;
    font-size: 13px;
}

.entry-table th,
.entry-table td {
    padding: 8px 10px;
    border-bottom: 1px solid #eef1f4;
    text-align: left;
}

.entry-table th {
    background: #f8fafc;
    color: #475569;
    font-size: 12px;
    font-weight: 700;
}

.row-action {
    text-align: right;
    width: 1%;
    white-space: nowrap;
}

.link-btn {
    padding: 0;
    background: none;
    border: none;
    color: #1f847f;
    font-family: inherit;
    font-size: 13px;
    text-decoration: underline;
    cursor: pointer;
    text-align: left;
}

.link-btn:hover {
    color: #14524f;
}

.check-grid {
    display: grid;
    grid-template-columns: repeat(2, minmax(0, 1fr));
    gap: 2px 16px;
}

.radio-column {
    display: flex;
    flex-direction: column;
    gap: 2px;
}

.choice-row {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 7px 10px;
    border-radius: 8px;
    font-size: 13px;
    color: #374151;
    cursor: pointer;
}

.choice-row:hover {
    background: #f8fafc;
}

.choice-row input {
    width: 16px;
    height: 16px;
    accent-color: #269c96;
    cursor: pointer;
    flex-shrink: 0;
}

.device-choice .device-label {
    display: flex;
    align-items: center;
    gap: 10px;
    cursor: pointer;
}

.dash {
    color: #6b7280;
}

.inline-choices {
    display: flex;
    gap: 12px;
    flex-wrap: wrap;
}

.add-history {
    display: grid;
    grid-template-columns: minmax(0, 1fr) minmax(0, 220px);
    gap: 16px 24px;
    align-items: start;
}

.age-column {
    display: flex;
    flex-direction: column;
    gap: 8px;
}

.botox-form {
    display: flex;
    flex-direction: column;
    gap: 10px;
}

.confirm-box {
    display: flex;
    flex-direction: column;
    gap: 8px;
    padding: 14px 16px;
    background: #fff9f0;
    border: 1px solid #f0dcc0;
    border-radius: 8px;
}

.confirm-actions {
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 16px;
}

.followup-input {
    max-width: 220px;
}

@media (max-width: 700px) {
    .check-grid,
    .add-history {
        grid-template-columns: 1fr;
    }
}















import { CommonModule } from '@angular/common';
import { ChangeDetectorRef, Component, EventEmitter, Input, OnChanges, Output, SimpleChanges } from '@angular/core';
import { FormsModule } from '@angular/forms';
import { HttpClient } from '@angular/common/http';

import { PODCI_BANK, PodciBankVariant, PodciBlock, PodciPage } from './podci-bank';

// PODCI (Pediatric Outcomes Data Collection Instrument), rendered directly from the legacy pages
// (clinconn/lab/q2/q2_{ch,ap,as}_001..027.jsp) via the generated PODCI_BANK — page text, option and
// scale labels, and answer values all come from the source, not from a hand-typed copy. The three
// variants are genuinely different pages (CH = parent-reported child, AP = parent-reported adolescent,
// AS = self-reported adolescent): different stems, different scales (AS has no "too young" options),
// and one fewer "limited by" checkbox on the AS participation follow-ups.
//
// Encodings, verified against the JSPs' radio/checkbox markup and save code:
//  - radios / scale rows store the option's 1-based value (value="1".."N"), not a 0-based index;
//  - the 16-condition grid (q2_007..022 a/b/c) is Yes=1 / No=2 radios; answering "No" to the first
//    (a) disables and leaves NULL the other two (b, c);
//  - checkboxes (body regions q2_023-025, "limited by" q2_062-098) store true/false — an unchecked
//    box on a page that was shown is saved as false, not omitted;
//  - the "limited by" follow-up pages are only shown for particular answers (decide_action() JS):
//    q2_061/069/077 -> only the 4th option ("No"); q2_085/091 -> only the 2nd or 3rd option;
//  - "[name]" in the text is the patient's first name (legacy currFName).
//
// Takes visitId (resolved once at picker time) — see history-visit-questionnaire.ts.

export type PodciVariant = PodciBankVariant;

interface FollowUp {
    followUpPage: number;
    key: string;
    showFor: number[];
}

// Legacy page number of the participation question -> its follow-up page, answer field, and which
// 1-based answer values open the follow-up.
const FOLLOW_UPS: Record<number, FollowUp> = {
    13: { followUpPage: 14, key: 'q2_061', showFor: [4] },
    15: { followUpPage: 16, key: 'q2_069', showFor: [4] },
    17: { followUpPage: 18, key: 'q2_077', showFor: [4] },
    19: { followUpPage: 20, key: 'q2_085', showFor: [2, 3] },
    21: { followUpPage: 22, key: 'q2_091', showFor: [2, 3] }
};
const FOLLOW_UP_PAGES = new Set(Object.values(FOLLOW_UPS).map(f => f.followUpPage));

type Answer = number | boolean | string | null;

export interface RenderedPodciPage {
    n: number;
    blocks: PodciBlock[];
}

@Component({
    selector: 'app-podci-questionnaire',
    standalone: true,
    imports: [CommonModule, FormsModule],
    templateUrl: './podci-questionnaire.html',
    styleUrl: './podci-questionnaire.css'
})
export class PodciQuestionnaire implements OnChanges {
    @Input() patientId: number | null = null;
    @Input() visitId: number | null = null;
    @Input() firstName = '';
    @Input() variant: PodciVariant = 'CH';
    @Input() language: 'en' | 'sp' = 'en';
    @Input() hideActions = false;
    @Output() saveComplete = new EventEmitter<void>();
    @Output() saveFailed = new EventEmitter<string>();

    answers: Record<string, Answer> = {};

    saving = false;
    saveError = '';
    saved = false;

    constructor(private http: HttpClient, private cdr: ChangeDetectorRef) {}

    // Only a change of what is being loaded resets the form — not e.g. the parent's live `copyWho`/`firstName`
    // updates, which would otherwise wipe whatever the user has already entered.
    ngOnChanges(changes: SimpleChanges): void {
        if (!(changes['patientId'] || changes['visitId'] || changes['variant'])) return;
        this.answers = {};
        this.saved = false;
        this.saveError = '';
        this.preload();
    }

    // Stored columns are q2_XXX -> raw number / boolean / text. Radios & scale are 1-based, the grid
    // a-part is Yes=1 / No=2 and checkboxes are booleans — exactly what buildAnswers() sends.
    private preload(): void {
        if (!this.patientId || !this.visitId) {
            return;
        }
        const visitId = this.visitId;
        const variant = this.variant;
        this.http.get<Record<string, Answer>>(
            `/api/patients/${this.patientId}/podci-questionnaire/${visitId}`,
            { params: { variant: variant.toLowerCase() } }
        ).subscribe({
            next: saved => {
                if (visitId !== this.visitId || variant !== this.variant) return;
                const loaded: Record<string, Answer> = {};
                for (const [key, value] of Object.entries(saved ?? {})) {
                    if (value !== null && value !== undefined) loaded[key] = value;
                }
                this.answers = { ...loaded, ...this.answers };
                this.cdr.markForCheck();
            },
            error: () => {}
        });
    }

    // Spanish exists only on the CH pages; AP/AS fall back to English.
    private blocksFor(page: PodciPage): PodciBlock[] {
        return (this.language === 'sp' && page.blocks.sp) ? page.blocks.sp : page.blocks.en;
    }

    // Pages in legacy order (page 0 is the variant's introduction), follow-up pages only when their
    // trigger answer was chosen. The page title (first text block, repeated on every legacy page)
    // is kept on page 1 only.
    get pages(): RenderedPodciPage[] {
        const out: RenderedPodciPage[] = [];
        for (const page of PODCI_BANK[this.variant]) {
            if (FOLLOW_UP_PAGES.has(page.n) && !this.followUpShown(page.n)) {
                continue;
            }
            const blocks = this.blocksFor(page);
            out.push({ n: page.n, blocks: page.n > 1 && blocks[0]?.t === 'text' ? blocks.slice(1) : blocks });
        }
        return out;
    }

    private followUpShown(pageNumber: number): boolean {
        const parent = Object.entries(FOLLOW_UPS).find(([, f]) => f.followUpPage === pageNumber);
        if (!parent) return true;
        const value = this.answers[parent[1].key];
        return typeof value === 'number' && parent[1].showFor.includes(value);
    }

    text(template: string): string {
        return template.replace(/\[name\]/g, this.firstName || (this.language === 'sp' ? 'su niño(a)' : 'your child'));
    }

    // --- answers -------------------------------------------------------------------------------
    selected(name: string, value: number): boolean {
        return this.answers[name] === value;
    }

    setRadio(name: string, value: number): void {
        this.answers[name] = value;
    }

    // Condition grid: "No" to the first question (…a) disables and clears the other two (…b, …c).
    setGrid(name: string, value: number): void {
        this.answers[name] = value;
        if (name.endsWith('a') && value === 2) {
            const base = name.slice(0, -1);
            this.answers[base + 'b'] = null;
            this.answers[base + 'c'] = null;
        }
    }

    gridDisabled(name: string): boolean {
        return /[bc]$/.test(name) && this.answers[name.slice(0, -1) + 'a'] === 2;
    }

    isChecked(name: string): boolean {
        return this.answers[name] === true;
    }

    setChecked(name: string, checked: boolean): void {
        this.answers[name] = checked;
    }

    comment(name: string): string {
        return (this.answers[name] as string) ?? '';
    }

    setComment(name: string, value: string): void {
        this.answers[name] = value;
    }

    // --- save ----------------------------------------------------------------------------------
    // Builds the payload the way legacy's pages would have stored it: every control on a shown page
    // is saved — unchecked checkboxes as false, an empty comment as '' — while unanswered radios
    // stay absent (NULL) and the b/c cells of a "No" grid row are explicit NULLs.
    private buildAnswers(): Record<string, Answer> {
        const out: Record<string, Answer> = {};
        for (const page of this.pages) {
            for (const block of page.blocks) {
                switch (block.t) {
                    case 'radios':
                        if (this.answers[block.name] != null) out[block.name] = this.answers[block.name];
                        break;
                    case 'scale':
                        for (const row of block.rows) {
                            if (this.answers[row.name] != null) out[row.name] = this.answers[row.name];
                        }
                        break;
                    case 'grid':
                        for (const row of block.rows) {
                            for (const cell of row.cells) {
                                if (this.gridDisabled(cell.name)) out[cell.name] = null;
                                else if (this.answers[cell.name] != null) out[cell.name] = this.answers[cell.name];
                            }
                        }
                        break;
                    case 'checks':
                        for (const item of block.items) out[item.name] = this.answers[item.name] === true;
                        break;
                    case 'textarea':
                        out[block.name] = (this.answers[block.name] as string) ?? '';
                        break;
                }
            }
        }
        return out;
    }

    // Legacy check_ok() (page-level, patient mode): every listed radio group must have a choice, or be
    // disabled (the b/c parts of a condition row whose a-part is "No"). Pages whose own check_ok() is
    // broken in the JSPs (q2_*_005, q2_ch/ap_011) carry no required list. Legacy skips all of this when
    // the session attribute currQuest == "clinician" (set by visits.jsp), which has no equivalent here.
    validate(): string | null {
        for (const page of this.pages) {
            const bankPage = PODCI_BANK[this.variant].find(p => p.n === page.n);
            for (const name of bankPage?.required ?? []) {
                if (this.gridDisabled(name)) continue;
                const value = this.answers[name];
                if (value === null || value === undefined) {
                    return 'Please respond to all questions before continuing.';
                }
            }
        }
        return null;
    }

    save(): void {
        if (!this.patientId || !this.visitId) {
            this.saveComplete.emit();
            return;
        }

        this.saving = true;
        this.saveError = '';

        this.http.post(
            `/api/patients/${this.patientId}/podci-questionnaire`,
            {
                visitId: this.visitId,
                variant: this.variant.toLowerCase(),
                answers: this.buildAnswers()
            }
        ).subscribe({
            next: () => {
                this.saving = false;
                this.saved = true;
                this.cdr.markForCheck();
                this.saveComplete.emit();
            },
            error: error => {
                console.error('Unable to save PODCI questionnaire:', error);
                this.saving = false;
                this.saveError = 'Unable to save. Please try again.';
                this.cdr.markForCheck();
                this.saveFailed.emit(this.saveError);
            }
        });
    }
}












// GENERATED from the legacy PODCI JSPs (clinconn/lab/q2/q2_{ch,ap,as}_001..027.jsp) by a script that
// parses each page's DOM: page text, option/scale labels, radio values, checkbox names. Do not hand-edit
// wording here — regenerate from source instead. English = the JSP's English branch; Spanish exists only
// for the CH (parent-reported child) pages, which switch language per session; AP/AS are English-only in
// legacy (their Spanish falls back to English). "[name]" is the patient's first name (currFName).
export type PodciLang = 'en' | 'sp';
export interface PodciOption { value: number; label: string; }
export type PodciBlock =
  | { t: 'text'; text: string }
  | { t: 'radios'; name: string; options: PodciOption[] }
  | { t: 'scale'; columns: string[]; rows: { name: string; text: string }[] }
  | { t: 'grid'; columns: string[]; rows: { label: string; cells: { name: string; options: PodciOption[] }[] }[] }
  | { t: 'checks'; items: { name: string; label: string; rowLabel: string }[]}
  | { t: 'textarea'; name: string };
export interface PodciPage { n: number; blocks: { en: PodciBlock[]; sp?: PodciBlock[] }; required?: string[]; }
export type PodciBankVariant = 'CH' | 'AP' | 'AS';

export const PODCI_BANK: Record<PodciBankVariant, PodciPage[]> = {
  CH: [
    { n: 0, blocks: { en: [{"t":"text","text":"The following is a survey designed by a group of doctors [1] to measure the quality of life of the children they treat."},{"t":"text","text":"Your answers will help us to understand how your child's condition affects his or her daily functions and overall happiness."},{"t":"text","text":"It will also let us see if the treatment your child receives improves his or her quality of life."},{"t":"text","text":"We appreciate your input and thank you for allowing us to provide this service."},{"t":"text","text":"[1] The Pediatric Outcomes Data Collection Instrument (PODCI), Version 2.0, revised Aug 2005."}], sp: [{"t":"text","text":"Le pedimos que complete esta encuesta sobre [name], a fin de entender mejor su salud en general, así como problemas relacionados con condiciones de los huesos y de los músculos. El completar esta encuesta es voluntario. Sus respuestas se mantendrán en la más estricta confidencialidad. Se tomará de 15 a 20 minutos para completarse."},{"t":"text","text":"Por favor, conteste cada una de las preguntas. Es posible que algunas preguntas se parezcan a otras, pero cada una es diferente."},{"t":"text","text":"Conteste las preguntas, poniendo un círculo en el número apropiado, o escribiendo la respuesta, según se requiera."},{"t":"text","text":"No hay respuestas correctas, ni incorrectas. Si no está seguro sobre cómo contestar una pregunta, por favor, dé la mejor respuesta que pueda dar, y escriba un comentario en el margen. Leeremos todos sus comentarios, así que siéntase en libertad de hacer tantos comentarios como quiera."},{"t":"text","text":"[1] The Pediatric Outcomes Data Collection Instrument (PODCI), Version 2.0, revised Aug 2005."}] } },
    { n: 1, blocks: { en: [{"t":"text","text":"Pediatric Health Assessment (parent reported)"},{"t":"text","text":"In general, would you say [name]'s health is:"},{"t":"radios","name":"q2_005","options":[{"value":1,"label":"Excellent"},{"value":2,"label":"Very Good"},{"value":3,"label":"Good"},{"value":4,"label":"Fair"},{"value":5,"label":"Poor"}]}], sp: [{"t":"text","text":"Evaluación de Salud Pediátrica (informado por los padres)"},{"t":"text","text":"En general, diría que la salud de [name] es:"},{"t":"radios","name":"q2_005","options":[{"value":1,"label":"Excelente"},{"value":2,"label":"Muy buena"},{"value":3,"label":"Buena"},{"value":4,"label":"Regular"},{"value":5,"label":"Mala"}]}] } },
    { n: 2, blocks: { en: [{"t":"text","text":"Pediatric Health Assessment (parent reported)"},{"t":"text","text":"Compared to one year ago, how would you rate [name]'s health in general:"},{"t":"radios","name":"q2_006","options":[{"value":1,"label":"Much better now than one year ago"},{"value":2,"label":"Somewhat better now than one year ago"},{"value":3,"label":"About the same as one year ago"},{"value":4,"label":"Somewhat worse now than one year ago"},{"value":5,"label":"Much worse now than one year ago"}]}], sp: [{"t":"text","text":"Evaluación de Salud Pediátrica (informado por los padres)"},{"t":"text","text":"Comparado con un año atrás, ¿cómo calificaría la salud de [name] en general ahora?"},{"t":"radios","name":"q2_006","options":[{"value":1,"label":"Mucho mejor que hace un año"},{"value":2,"label":"Algo mejor que hace un año"},{"value":3,"label":"Casi igual"},{"value":4,"label":"Algo peor que hace un año"},{"value":5,"label":"Mucho peor que hace un año"}]}] } },
    { n: 3, blocks: { en: [{"t":"text","text":"Pediatric Health Assessment (parent reported)"},{"t":"text","text":"Have you ever been told by a doctor, nurse, teacher, or other health professional that [name] has had any of the following conditions?"},{"t":"text","text":"If yes, indicate if your child is being treated for this condition and if your child is limited by those conditions."},{"t":"grid","columns":["Has your child ever had it?","Does your child receive treatment for it now?","Are your child's activities limited by it now?"],"rows":[{"label":"Juvenile Arthritis (one or two joints)","cells":[{"name":"q2_007a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_007b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_007c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Juvenile Arthritis (many joints)","cells":[{"name":"q2_008a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_008b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_008c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Anorexia or bulemia (eating disorders)","cells":[{"name":"q2_009a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_009b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_009c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Asthma","cells":[{"name":"q2_010a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_010b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_010c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Attention or behavioral problems","cells":[{"name":"q2_011a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_011b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_011c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Chronic allergies or sinus trouble","cells":[{"name":"q2_012a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_012b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_012c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Developmental delay","cells":[{"name":"q2_013a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_013b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_013c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Cognitive Impairment","cells":[{"name":"q2_014a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_014b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_014c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Diabetes","cells":[{"name":"q2_015a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_015b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_015c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Epilepsy (seizure disorder)","cells":[{"name":"q2_016a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_016b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_016c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Hearing impairment or deafness","cells":[{"name":"q2_017a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_017b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_017c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Heart problem","cells":[{"name":"q2_018a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_018b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_018c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Learning problem","cells":[{"name":"q2_019a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_019b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_019c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Sleep disturbance","cells":[{"name":"q2_020a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_020b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_020c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Speech problems","cells":[{"name":"q2_021a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_021b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_021c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Vision problems","cells":[{"name":"q2_022a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_022b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_022c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]}]}], sp: [{"t":"text","text":"Evaluación de Salud Pediátrica (informado por los padres)"},{"t":"text","text":"¿Alguna vez le ha dicho un(a) doctor(a), enfermero(a), maestro(a) u otro profesional médico que [name] ha tenido alguna de las siguientes condiciones?"},{"t":"text","text":"Si la respuesta es sí, indique si a su niño(a) se le está tratando para esta condición, y si su niño(a) está limitado(a) a causa de estas condiciones."},{"t":"grid","columns":["¿La ha tenido [name] alguna vez?","¿Recibe [name] tratamiento para ésta ahora?","¿Están limitadas las actividades de [name] por ésta ahora?"],"rows":[{"label":"Artritis juvenil (una o dos articulaciones)","cells":[{"name":"q2_007a","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]},{"name":"q2_007b","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]},{"name":"q2_007c","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]}]},{"label":"Artritis juvenil (varias articulaciones)","cells":[{"name":"q2_008a","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]},{"name":"q2_008b","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]},{"name":"q2_008c","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]}]},{"label":"Anorexia o bulimia (Problemas de alimentación)","cells":[{"name":"q2_009a","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]},{"name":"q2_009b","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]},{"name":"q2_009c","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]}]},{"label":"Asma","cells":[{"name":"q2_010a","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]},{"name":"q2_010b","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]},{"name":"q2_010c","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]}]},{"label":"Problemas de atención o comportamiento","cells":[{"name":"q2_011a","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]},{"name":"q2_011b","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]},{"name":"q2_011c","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]}]},{"label":"Alergias crónicas o problemas de sinusitis","cells":[{"name":"q2_012a","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]},{"name":"q2_012b","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]},{"name":"q2_012c","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]}]},{"label":"Retraso del desarrollo","cells":[{"name":"q2_013a","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]},{"name":"q2_013b","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]},{"name":"q2_013c","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]}]},{"label":"Retraso mental","cells":[{"name":"q2_014a","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]},{"name":"q2_014b","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]},{"name":"q2_014c","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]}]},{"label":"Diabetes","cells":[{"name":"q2_015a","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]},{"name":"q2_015b","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]},{"name":"q2_015c","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]}]},{"label":"Epilepsia (ataques)","cells":[{"name":"q2_016a","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]},{"name":"q2_016b","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]},{"name":"q2_016c","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]}]},{"label":"Problemas del oído o sordera","cells":[{"name":"q2_017a","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]},{"name":"q2_017b","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]},{"name":"q2_017c","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]}]},{"label":"Problemas del corazón","cells":[{"name":"q2_018a","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]},{"name":"q2_018b","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]},{"name":"q2_018c","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]}]},{"label":"Problemas de aprendizaje","cells":[{"name":"q2_019a","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]},{"name":"q2_019b","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]},{"name":"q2_019c","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]}]},{"label":"Trastornos del sueño","cells":[{"name":"q2_020a","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]},{"name":"q2_020b","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]},{"name":"q2_020c","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]}]},{"label":"Problemas del habla","cells":[{"name":"q2_021a","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]},{"name":"q2_021b","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]},{"name":"q2_021c","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]}]},{"label":"Problemas de la vista","cells":[{"name":"q2_022a","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]},{"name":"q2_022b","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]},{"name":"q2_022c","options":[{"value":1,"label":"Sí"},{"value":2,"label":"No"}]}]}]}] }, required: ["q2_007a","q2_008a","q2_009a","q2_010a","q2_011a","q2_012a","q2_013a","q2_014a","q2_015a","q2_016a","q2_017a","q2_018a","q2_019a","q2_020a","q2_021a","q2_022a","q2_007b","q2_008b","q2_009b","q2_010b","q2_011b","q2_012b","q2_013b","q2_014b","q2_015b","q2_016b","q2_017b","q2_018b","q2_019b","q2_020b","q2_021b","q2_022b","q2_007c","q2_008c","q2_009c","q2_010c","q2_011c","q2_012c","q2_013c","q2_014c","q2_015c","q2_016c","q2_017c","q2_018c","q2_019c","q2_020c","q2_021c","q2_022c"] },
    { n: 4, blocks: { en: [{"t":"text","text":"Pediatric Health Assessment (parent reported)"},{"t":"text","text":"For [name]'s front right side , please indicate those areas that bother him/her or limit his/her function."},{"t":"checks","items":[{"name":"q2_023a","label":"Neck","rowLabel":""},{"name":"q2_023b","label":"Shoulder area","rowLabel":""},{"name":"q2_023c","label":"Elbow/Forearm","rowLabel":""},{"name":"q2_023d","label":"Wrist/Hand","rowLabel":""},{"name":"q2_023e","label":"Hip","rowLabel":""},{"name":"q2_023f","label":"Thigh","rowLabel":""},{"name":"q2_023g","label":"Knee area","rowLabel":""},{"name":"q2_023h","label":"Calf area","rowLabel":""},{"name":"q2_023i","label":"Ankle/Foot area","rowLabel":""}]},{"t":"text","text":"For [name]'s front left side , please indicate those areas that bother him/her or limit his/her function."},{"t":"checks","items":[{"name":"q2_024a","label":"Neck","rowLabel":""},{"name":"q2_024b","label":"Shoulder area","rowLabel":""},{"name":"q2_024c","label":"Elbow/Forearm","rowLabel":""},{"name":"q2_024d","label":"Wrist/Hand","rowLabel":""},{"name":"q2_024e","label":"Hip","rowLabel":""},{"name":"q2_024f","label":"Thigh","rowLabel":""},{"name":"q2_024g","label":"Knee area","rowLabel":""},{"name":"q2_024h","label":"Calf area","rowLabel":""},{"name":"q2_024i","label":"Ankle/Foot area","rowLabel":""}]},{"t":"text","text":"For [name]'s back , please indicate those areas that bother him/her or limit his/her function."},{"t":"checks","items":[{"name":"q2_025a","label":"Neck","rowLabel":""},{"name":"q2_025b","label":"Upper Back","rowLabel":""},{"name":"q2_025c","label":"Lower Back","rowLabel":""}]}], sp: [{"t":"text","text":"Evaluación de Salud Pediátrica (informado por los padres)"},{"t":"text","text":"Para el lado derecho del cuerpo, por favor indique aquellas áreas que le producen molestias a [name] o limitan su funcionamiento."},{"t":"checks","items":[{"name":"q2_023a","label":"Cuello","rowLabel":""},{"name":"q2_023b","label":"Área del Hombro","rowLabel":""},{"name":"q2_023c","label":"Codo/Antebrazo","rowLabel":""},{"name":"q2_023d","label":"Muñeca/Mano","rowLabel":""},{"name":"q2_023e","label":"Cadera","rowLabel":""},{"name":"q2_023f","label":"Muslo","rowLabel":""},{"name":"q2_023g","label":"Área de la Rodilla","rowLabel":""},{"name":"q2_023h","label":"Área de la Pantorilla","rowLabel":""},{"name":"q2_023i","label":"Área del Tobillo/Pie","rowLabel":""}]},{"t":"text","text":"Para el lado izquierdo del cuerpo, por favor indique aquellas áreas que le producen molestias a [name] o limitan su funcionamiento."},{"t":"checks","items":[{"name":"q2_024a","label":"Cuello","rowLabel":""},{"name":"q2_024b","label":"Área del Hombro","rowLabel":""},{"name":"q2_024c","label":"Codo/Antebrazo","rowLabel":""},{"name":"q2_024d","label":"Muñeca/Mano","rowLabel":""},{"name":"q2_024e","label":"Cadera","rowLabel":""},{"name":"q2_024f","label":"Muslo","rowLabel":""},{"name":"q2_024g","label":"Área de la Rodilla","rowLabel":""},{"name":"q2_024h","label":"Área de la Pantorilla","rowLabel":""},{"name":"q2_024i","label":"Área del Tobillo/Pie","rowLabel":""}]},{"t":"text","text":"En la parte de la espalda , por favor indique aquellas áreas que le producen molestias a [name] o limitan su funcionamiento"},{"t":"checks","items":[{"name":"q2_025a","label":"Cuello","rowLabel":""},{"name":"q2_025b","label":"Espalda alta","rowLabel":""},{"name":"q2_025c","label":"Espalda baja","rowLabel":""}]}] } },
    { n: 5, blocks: { en: [{"t":"text","text":"Pediatric Health Assessment (parent reported)"},{"t":"text","text":"Some kinds of problems can make it hard to do many activities, such as eating, bathing, school work, and playing with friends."},{"t":"text","text":"We would like to find out how [name] is doing."},{"t":"text","text":"During the last week was it easy or hard for your child to:"},{"t":"scale","columns":["Easy","A little hard","Very hard","Can't do at all","Too young for this activity"],"rows":[{"name":"q2_026","text":"Lift heavy books?"},{"name":"q2_027","text":"Pour a half gallon of milk?"},{"name":"q2_028","text":"Open a jar that has been opened before?"},{"name":"q2_029","text":"Use a fork and spoon?"},{"name":"q2_030","text":"Comb his/her hair?"},{"name":"q2_031","text":"Button buttons?"},{"name":"q2_032x","text":"Put on his/her coat?"},{"name":"q2_033","text":"Write with a pencil?"}]}], sp: [{"t":"text","text":"Evaluación de Salud Pediátrica (informado por los padres)"},{"t":"text","text":"Algunas tipos de problemas pueden dificultar realizar muchas actividades, como comer, bañarse, tareas escolares y jugar con los amigos."},{"t":"text","text":"Nos gustaría saber cómo está [name]."},{"t":"text","text":"Durante la semana pasada ¿fue fácil o difícil para [name]:"},{"t":"scale","columns":["Fácil","Un poco difícil","Muy difícil","No puede hacerlo","Muy joven para esta actividad"],"rows":[{"name":"q2_026","text":"levantar libros pesados?"},{"name":"q2_027","text":"servir un medio galón de leche?"},{"name":"q2_028","text":"abrir un frasco que se ha abierto anteriormente?"},{"name":"q2_029","text":"usar un tenedor y una cuchara?"},{"name":"q2_030","text":"peinarse el pelo?"},{"name":"q2_031","text":"abrocharse los botones?"},{"name":"q2_032x","text":"ponerse los calcetines?"},{"name":"q2_033","text":"escribir con un lápiz?"}]}] } },
    { n: 6, blocks: { en: [{"t":"text","text":"Pediatric Health Assessment (parent reported)"},{"t":"text","text":"On average, over the last 12 months , how often did [name] miss school (preschool, day care, camp, etc.) because of his/her health?"},{"t":"radios","name":"q2_034","options":[{"value":1,"label":"Rarely"},{"value":2,"label":"Once a month"},{"value":3,"label":"Two or three times a month"},{"value":4,"label":"Once a week"},{"value":5,"label":"More than once a week"},{"value":6,"label":"Does not attend school, etc."}]}], sp: [{"t":"text","text":"Evaluación de Salud Pediátrica (informado por los padres)"},{"t":"text","text":"En un promedio, durante los últimos 12 meses , ¿con qué frecuencia faltó [name] a la escuela (kinder, guardería, campamento, etc.) debido a su salud?"},{"t":"radios","name":"q2_034","options":[{"value":1,"label":"Rara vez"},{"value":2,"label":"Una vez al mes"},{"value":3,"label":"Dos o tres veces al mes"},{"value":4,"label":"Una vez a la semana"},{"value":5,"label":"Más de una vez a la semana"},{"value":6,"label":"No va a la escuela, etc."}]}] }, required: ["q2_034"] },
    { n: 7, blocks: { en: [{"t":"text","text":"Pediatric Health Assessment (parent reported)"},{"t":"text","text":"During the last week , how happy has [name] been with:"},{"t":"scale","columns":["Very happy","Somewhat happy","Not sure","Somewhat unhappy","Very unhappy","Child is too young"],"rows":[{"name":"q2_035","text":"How he/she looks?"},{"name":"q2_036","text":"His/her body?"},{"name":"q2_037","text":"What clothes or shoes he/she can wear"},{"name":"q2_038","text":"His/her ability to do the same things his/her friends do?"},{"name":"q2_039","text":"His/her health in general?"}]}], sp: [{"t":"text","text":"Evaluación de Salud Pediátrica (informado por los padres)"},{"t":"text","text":"Durante la semana pasada , ¿qué tan feliz ha estado [name] con:"},{"t":"scale","columns":["Muy feliz","Algo feliz","No estoy seguro(a)","Algo infeliz","Muy infeliz","El niño(a) es muy joven"],"rows":[{"name":"q2_035","text":"cómo se ve?"},{"name":"q2_036","text":"su cuerpo?"},{"name":"q2_037","text":"la ropa o zapatos que puede usar?"},{"name":"q2_038","text":"su habilidad para hacer las mismas cosas que hacen sus amigos?"},{"name":"q2_039","text":"su salud en general?"}]}] }, required: ["q2_035","q2_036","q2_037","q2_038","q2_039"] },
    { n: 8, blocks: { en: [{"t":"text","text":"Pediatric Health Assessment (parent reported)"},{"t":"text","text":"During the last week , how much of the time:"},{"t":"scale","columns":["Most of the time","Some of the time","A little of the time","None of the time"],"rows":[{"name":"q2_040","text":"Did [name] feel sick and tired?"},{"name":"q2_041","text":"Was [name] full of pep and energy?"},{"name":"q2_042","text":"Did pain or discomfort interfere with [name]'s activities?"}]}], sp: [{"t":"text","text":"Evaluación de Salud Pediátrica (informado por los padres)"},{"t":"text","text":"Durante la semana pasada , ¿cuánto tiempo:"},{"t":"scale","columns":["La mayoría del tiempo","Parte del tiempo","Poco tiempo","En ningún momento"],"rows":[{"name":"q2_040","text":"se sintió [name] mal y cansado(a)?"},{"name":"q2_041","text":"se sintió [name] lleno(a) de vigor y energía?"},{"name":"q2_042","text":"interfirió el dolor o malestar con las actividades de [name]?"}]}] }, required: ["q2_040","q2_041","q2_042"] },
    { n: 9, blocks: { en: [{"t":"text","text":"Pediatric Health Assessment (parent reported)"},{"t":"text","text":"During the last week , has it been easy or hard for [name] to:"},{"t":"scale","columns":["Easy","A little hard","Very hard","Can't do at all","Too young for this activity"],"rows":[{"name":"q2_043","text":"Run short distances?"},{"name":"q2_044","text":"Bicycle or tricycle?"},{"name":"q2_045","text":"Climb three flights of stairs?"},{"name":"q2_046","text":"Climb one flight of stairs?"},{"name":"q2_047","text":"Walk more than a mile?"},{"name":"q2_048","text":"Walk three blocks?"},{"name":"q2_049","text":"Walk one block?"},{"name":"q2_050","text":"Get on and off a bus?"}]}], sp: [{"t":"text","text":"Evaluación de Salud Pediátrica (informado por los padres)"},{"t":"text","text":"Durante la semana pasada , ¿ha sido fácil o difícil para [name]:"},{"t":"scale","columns":["Fácil","Un poco difícil","Muy difícil","No puede hacerlo para nada","Muy joven para esta actividad"],"rows":[{"name":"q2_043","text":"correr distancias cortas?"},{"name":"q2_044","text":"andar en bicicleta o triciclo?"},{"name":"q2_045","text":"subir tres tramos de escaleras?"},{"name":"q2_046","text":"subir un tramo de escaleras?"},{"name":"q2_047","text":"caminar más de una milla?"},{"name":"q2_048","text":"caminar tres cuadras?"},{"name":"q2_049","text":"caminar una cuadra?"},{"name":"q2_050","text":"subir o bajar del autobús?"}]}] }, required: ["q2_043","q2_044","q2_045","q2_046","q2_047","q2_048","q2_049","q2_050"] },
    { n: 10, blocks: { en: [{"t":"text","text":"Pediatric Health Assessment (parent reported)"},{"t":"text","text":"How often does [name] need help from another person for walking and climbing?"},{"t":"radios","name":"q2_051","options":[{"value":1,"label":"Never"},{"value":2,"label":"Sometimes"},{"value":3,"label":"About half the time"},{"value":4,"label":"Often"},{"value":5,"label":"All the time"}]},{"t":"text","text":"How often does [name] use assistive devices (such as braces, crutches, or wheelchair) for walking and climbing?"},{"t":"radios","name":"q2_052","options":[{"value":1,"label":"Never"},{"value":2,"label":"Sometimes"},{"value":3,"label":"About half the time"},{"value":4,"label":"Often"},{"value":5,"label":"All the time"}]}], sp: [{"t":"text","text":"Evaluación de Salud Pediátrica (informado por los padres)"},{"t":"text","text":"¿Con qué frecuencia necesita [name] ayuda de otra persona para caminar o subir?"},{"t":"radios","name":"q2_051","options":[{"value":1,"label":"Nunca"},{"value":2,"label":"Algunas veces"},{"value":3,"label":"Aproximadamente la mitad del tiempo"},{"value":4,"label":"Frecuentemente"},{"value":5,"label":"Todo el tiempo"}]},{"t":"text","text":"¿Con qué frecuencia usa [name] aparatos auxiliares (como férulas, muletas, andador o sillas de ruedas para caminar o subir?"},{"t":"radios","name":"q2_052","options":[{"value":1,"label":"Nunca"},{"value":2,"label":"Algunas veces"},{"value":3,"label":"Aproximadamente la mitad del tiempo"},{"value":4,"label":"Frecuentemente"},{"value":5,"label":"Todo el tiempo"}]}] }, required: ["q2_051","q2_052"] },
    { n: 11, blocks: { en: [{"t":"text","text":"Pediatric Health Assessment (parent reported)"},{"t":"text","text":"During the last week , has it been easy or hard for [name] to:"},{"t":"scale","columns":["Easy","A little hard","Very hard","Can't do at all","Too young for this activity"],"rows":[{"name":"q2_053","text":"Stand while washing his/her hands and face at a sink?"},{"name":"q2_054","text":"Sit in a regular chair without holding on?"},{"name":"q2_055","text":"Get on and off a toilet or chair?"},{"name":"q2_056","text":"Get in and out of bed?"},{"name":"q2_057","text":"Turn door knobs?"},{"name":"q2_058","text":"Bend over from a standing position and pick up something off the floor?"}]}], sp: [{"t":"text","text":"Evaluación de Salud Pediátrica (informado por los padres)"},{"t":"text","text":"Durante la semana pasada , ¿ha sido fácil o difícil para [name]:"},{"t":"scale","columns":["Fácil","Un poco difícil","Muy difícil","No puede hacerlo para nada","Muy joven para esta actividad"],"rows":[{"name":"q2_053","text":"estar de pie mientras se lava las manos y cara ?"},{"name":"q2_054","text":"sentarse en una silla normal sin sujetarse de algo?"},{"name":"q2_055","text":"sentarse y pararse de un inodoro o silla?"},{"name":"q2_056","text":"acostarse o levantarse de la cama?"},{"name":"q2_057","text":"Dar vuelta las perillas de las puertas?"},{"name":"q2_058","text":"agacharse cuando está de pie para recoger algo del piso?"}]}] } },
    { n: 12, blocks: { en: [{"t":"text","text":"Pediatric Health Assessment (parent reported)"},{"t":"text","text":"How often does [name] need help from another person for sitting and standing?"},{"t":"radios","name":"q2_059","options":[{"value":1,"label":"Never"},{"value":2,"label":"Sometimes"},{"value":3,"label":"About half the time"},{"value":4,"label":"Often"},{"value":5,"label":"All the time"}]},{"t":"text","text":"How often does [name] use assistive devices (such as braces, crutches, or wheelchair) for sitting and standing?"},{"t":"radios","name":"q2_060","options":[{"value":1,"label":"Never"},{"value":2,"label":"Sometimes"},{"value":3,"label":"About half the time"},{"value":4,"label":"Often"},{"value":5,"label":"All the time"}]}], sp: [{"t":"text","text":"Evaluación de Salud Pediátrica (informado por los padres)"},{"t":"text","text":"¿Con qué frecuencia necesita [name] ayuda de otra persona para sentarse y ponerse de pie?"},{"t":"radios","name":"q2_059","options":[{"value":1,"label":"Nunca"},{"value":2,"label":"Algunas veces"},{"value":3,"label":"Aproximadamente la mitad del tiempo"},{"value":4,"label":"Frecuentemente"},{"value":5,"label":"Todo el tiempo"}]},{"t":"text","text":"¿Con qué frecuencia usa [name] la ayuda de aparatos auxiliares (como férulas, muletas, andador o sillas de ruedas) para sentarse o ponerse de pie?"},{"t":"radios","name":"q2_060","options":[{"value":1,"label":"Nunca"},{"value":2,"label":"Algunas veces"},{"value":3,"label":"Aproximadamente la mitad del tiempo"},{"value":4,"label":"Frecuentemente"},{"value":5,"label":"Todo el tiempo"}]}] }, required: ["q2_059","q2_060"] },
    { n: 13, blocks: { en: [{"t":"text","text":"Pediatric Health Assessment (parent reported)"},{"t":"text","text":"Can [name] participate in recreational outdoor activities with other children the same age?"},{"t":"text","text":"(For example: bicycling, tricycling, skating, hiking, jogging)"},{"t":"radios","name":"q2_061","options":[{"value":1,"label":"Yes, easily"},{"value":2,"label":"Yes, but a little hard"},{"value":3,"label":"Yes, but very hard"},{"value":4,"label":"No"}]}], sp: [{"t":"text","text":"Evaluación de Salud Pediátrica (informado por los padres)"},{"t":"text","text":"¿Puede [name] participar en actividades recreativas al aire libre con otros niños de su misma edad?"},{"t":"text","text":"(Por ejemplo: andar en bicicleta, triciclo, patines, escalar o trotar)"},{"t":"radios","name":"q2_061","options":[{"value":1,"label":"Sí, fácilmente"},{"value":2,"label":"Sí, pero es un poco difícil"},{"value":3,"label":"Sí, pero es muy difícil"},{"value":4,"label":"No"}]}] }, required: ["q2_061"] },
    { n: 14, blocks: { en: [{"t":"text","text":"Pediatric Health Assessment (parent reported)"},{"t":"text","text":"Can [name] participate in recreational outdoor activities with other children the same age? (cont.)"},{"t":"text","text":"(For example: bicycling, tricycling, skating, hiking, jogging)"},{"t":"text","text":"Was [name]'s activity limited by:"},{"t":"checks","items":[{"name":"q2_062","label":"Yes","rowLabel":"Pain?"},{"name":"q2_063","label":"Yes","rowLabel":"General Health?"},{"name":"q2_064","label":"Yes","rowLabel":"Doctor or parent instructions?"},{"name":"q2_065","label":"Yes","rowLabel":"Fear the other kids won't like him/her?"},{"name":"q2_066","label":"Yes","rowLabel":"Dislike of recreational outdoor activities?"},{"name":"q2_067","label":"Yes","rowLabel":"Too young?"},{"name":"q2_068","label":"Yes","rowLabel":"Activity not in season?"}]},{"t":"text","text":"Additional Comment for Selection(s) Above:"},{"t":"textarea","name":"q2_061comment"}], sp: [{"t":"text","text":"Evaluación de Salud Pediátrica (informado por los padres)"},{"t":"text","text":"¿Puede [name] participar en actividades recreativas al aire libre con otros niños de su misma edad?"},{"t":"text","text":"(Por ejemplo: andar en bicicleta, triciclo, patines, escalar o trotar)"},{"t":"text","text":"¿se limitó la actividad de [name] debido a:"},{"t":"checks","items":[{"name":"q2_062","label":"Sí","rowLabel":"dolor?"},{"name":"q2_063","label":"Sí","rowLabel":"salud en general?"},{"name":"q2_064","label":"Sí","rowLabel":"instrucciones del médico o de los padres?"},{"name":"q2_065","label":"Sí","rowLabel":"miedo a no ser aceptado por los otros niños?"},{"name":"q2_066","label":"Sí","rowLabel":"Desagrado a las actividades recreativas al aire libre?"},{"name":"q2_067","label":"Sí","rowLabel":"que es muy joven?"},{"name":"q2_068","label":"Sí","rowLabel":"que la actividad está fuera de temporada?"}]},{"t":"text","text":"Comentario Adicional para las Selecciones Anteriores:"},{"t":"textarea","name":"q2_061comment"}] } },
    { n: 15, blocks: { en: [{"t":"text","text":"Pediatric Health Assessment (parent reported)"},{"t":"text","text":"Can [name] participate in pickup games or sports with other children the same age?"},{"t":"text","text":"(For example: tag, dodge ball, basketball, soccer, catch, jump rope, touch football, hop scotch)"},{"t":"radios","name":"q2_069","options":[{"value":1,"label":"Yes, easily"},{"value":2,"label":"Yes, but a little hard"},{"value":3,"label":"Yes, but very hard"},{"value":4,"label":"No"}]}], sp: [{"t":"text","text":"Evaluación de Salud Pediátrica (informado por los padres)"},{"t":"text","text":"¿Puede [name] participar en juegos o deportes de pelota o de movimiento con otros niños de su misma edad?"},{"t":"text","text":"[Por ejemplo, la roña (tag), esquivar la pelota (dodge ball), baloncesto, fútbol soccer, atrapar la pelota (catch), saltar la cuerda, fútbol americano, rayuela (hop scotch)]"},{"t":"radios","name":"q2_069","options":[{"value":1,"label":"Sí, fácilmente"},{"value":2,"label":"Sí, pero es un poco difícil"},{"value":3,"label":"Sí, pero es muy difícil"},{"value":4,"label":"No"}]}] }, required: ["q2_069"] },
    { n: 16, blocks: { en: [{"t":"text","text":"Pediatric Health Assessment (parent reported)"},{"t":"text","text":"Can [name] participate in pickup games or sports with other children the same age? (cont.)"},{"t":"text","text":"(For example: tag, dodge ball, basketball, soccer, catch, jump rope, touch football, hop scotch)"},{"t":"text","text":"Was [name]'s activity limited by:"},{"t":"checks","items":[{"name":"q2_070","label":"Yes","rowLabel":"Pain?"},{"name":"q2_071","label":"Yes","rowLabel":"General Health?"},{"name":"q2_072","label":"Yes","rowLabel":"Doctor or parent instructions?"},{"name":"q2_073","label":"Yes","rowLabel":"Fear the other kids won't like him/her?"},{"name":"q2_074","label":"Yes","rowLabel":"Dislike of pickup games or sports?"},{"name":"q2_075","label":"Yes","rowLabel":"Too young?"},{"name":"q2_076","label":"Yes","rowLabel":"Activity not in season?"}]},{"t":"text","text":"Additional Comment for Selection(s) Above:"},{"t":"textarea","name":"q2_069comment"}], sp: [{"t":"text","text":"Evaluación de Salud Pediátrica (informado por los padres)"},{"t":"text","text":"¿Puede [name] participar en juegos o deportes de pelota o de movimiento con otros niños de su misma edad?"},{"t":"text","text":"[Por ejemplo, la roña (tag), esquivar la pelota (dodge ball), baloncesto, fútbol soccer, atrapar la pelota (catch), saltar la cuerda, fútbol americano, rayuela (hop scotch)]"},{"t":"text","text":"¿se limitó la actividad de [name] debido a:"},{"t":"checks","items":[{"name":"q2_070","label":"Sí","rowLabel":"dolor?"},{"name":"q2_071","label":"Sí","rowLabel":"salud en general?"},{"name":"q2_072","label":"Sí","rowLabel":"instrucciones del médico o de los padres?"},{"name":"q2_073","label":"Sí","rowLabel":"miedo a no ser aceptado por los otros niños?"},{"name":"q2_074","label":"Sí","rowLabel":"su disgusto por los juegos o deportes de pelota o de movimiento?"},{"name":"q2_075","label":"Sí","rowLabel":"que es muy joven?"},{"name":"q2_076","label":"Sí","rowLabel":"que la actividad está fuera de temporada?"}]},{"t":"text","text":"Comentario Adicional para las Selecciones Anteriores:"},{"t":"textarea","name":"q2_069comment"}] } },
    { n: 17, blocks: { en: [{"t":"text","text":"Pediatric Health Assessment (parent reported)"},{"t":"text","text":"Can [name] participate in competitive level sports with other children the same age?"},{"t":"text","text":"(For example: hockey, basketball, soccer, football, baseball, swimming, running [track or cross country], gymnastics, or dance)"},{"t":"radios","name":"q2_077","options":[{"value":1,"label":"Yes, easily"},{"value":2,"label":"Yes, but a little hard"},{"value":3,"label":"Yes, but very hard"},{"value":4,"label":"No"}]}], sp: [{"t":"text","text":"Evaluación de Salud Pediátrica (informado por los padres)"},{"t":"text","text":"¿Puede [name] participar en deportes a nivel competitivo con otros niños de su misma edad?"},{"t":"text","text":"[Por ejemplo: hockey, baloncesto, fútbol soccer, fútbol americano, béisbol, natación, carreras (carreras en pistas o a campo traviesa) (track o cross country)], gimnasia o danza]"},{"t":"radios","name":"q2_077","options":[{"value":1,"label":"Sí, fácilmente"},{"value":2,"label":"Sí, pero es un poco difícil"},{"value":3,"label":"Sí, pero es muy difícil"},{"value":4,"label":"No"}]}] }, required: ["q2_077"] },
    { n: 18, blocks: { en: [{"t":"text","text":"Pediatric Health Assessment (parent reported)"},{"t":"text","text":"Can [name] participate in competitive level sports with other children the same age? (cont.)"},{"t":"text","text":"(For example: hockey, basketball, soccer, football, baseball, swimming, running [track or cross country], gymnastics, or dance)"},{"t":"text","text":"Was [name]'s activity limited by:"},{"t":"checks","items":[{"name":"q2_078","label":"Yes","rowLabel":"Pain?"},{"name":"q2_079","label":"Yes","rowLabel":"General Health?"},{"name":"q2_080","label":"Yes","rowLabel":"Doctor or parent instructions?"},{"name":"q2_081","label":"Yes","rowLabel":"Fear the other kids won't like him/her?"},{"name":"q2_082","label":"Yes","rowLabel":"Dislike of competitive level sports?"},{"name":"q2_083","label":"Yes","rowLabel":"Too young?"},{"name":"q2_084","label":"Yes","rowLabel":"Activity not in season?"}]},{"t":"text","text":"Additional Comment for Selection(s) Above:"},{"t":"textarea","name":"q2_077comment"}], sp: [{"t":"text","text":"Evaluación de Salud Pediátrica (informado por los padres)"},{"t":"text","text":"¿Puede [name] participar en deportes a nivel competitivo con otros niños de su misma edad?"},{"t":"text","text":"[Por ejemplo: hockey, baloncesto, fútbol soccer, fútbol americano, béisbol, natación, carreras (carreras en pistas o a campo traviesa) (track o cross country)], gimnasia o danza]"},{"t":"text","text":"¿se limitó la actividad de [name] debido a:"},{"t":"checks","items":[{"name":"q2_078","label":"Sí","rowLabel":"dolor?"},{"name":"q2_079","label":"Sí","rowLabel":"salud en general?"},{"name":"q2_080","label":"Sí","rowLabel":"instrucciones del médico o de los padres?"},{"name":"q2_081","label":"Sí","rowLabel":"miedo a no ser aceptado por los otros niños?"},{"name":"q2_082","label":"Sí","rowLabel":"su disgusto por los deportes a nivel competitivo?"},{"name":"q2_083","label":"Sí","rowLabel":"que es muy joven?"},{"name":"q2_084","label":"Sí","rowLabel":"que la actividad está fuera de temporada?"}]},{"t":"text","text":"Comentario Adicional para las Selecciones Anteriores:"},{"t":"textarea","name":"q2_077comment"}] } },
    { n: 19, blocks: { en: [{"t":"text","text":"Pediatric Health Assessment (parent reported)"},{"t":"text","text":"How often in the last week did [name] get together and do things with friends?"},{"t":"radios","name":"q2_085","options":[{"value":1,"label":"Often"},{"value":2,"label":"Sometimes"},{"value":3,"label":"Never or rarely"}]}], sp: [{"t":"text","text":"Evaluación de Salud Pediátrica (informado por los padres)"},{"t":"text","text":"¿Con qué frecuencia durante la semana pasada ¿[name] se reunió o hizo algo con sus amigos?"},{"t":"radios","name":"q2_085","options":[{"value":1,"label":"Frecuentemente"},{"value":2,"label":"Algunas veces"},{"value":3,"label":"Nunca o rara vez"}]}] }, required: ["q2_085"] },
    { n: 20, blocks: { en: [{"t":"text","text":"Pediatric Health Assessment (parent reported)"},{"t":"text","text":"How often in the last week did [name] get together and do things with friends? (cont.)"},{"t":"text","text":"Was [name]'s activity limited by:"},{"t":"checks","items":[{"name":"q2_086","label":"Yes","rowLabel":"Pain?"},{"name":"q2_087","label":"Yes","rowLabel":"General Health?"},{"name":"q2_088","label":"Yes","rowLabel":"Doctor or parent instructions?"},{"name":"q2_089","label":"Yes","rowLabel":"Fear the other kids won't like him/her?"},{"name":"q2_090","label":"Yes","rowLabel":"Friends not around?"}]},{"t":"text","text":"Additional Comment for Selection(s) Above:"},{"t":"textarea","name":"q2_085comment"}], sp: [{"t":"text","text":"Evaluación de Salud Pediátrica (informado por los padres)"},{"t":"text","text":"¿Con qué frecuencia durante la semana pasada ¿[name] se reunió o hizo algo con sus amigos?"},{"t":"text","text":"¿se limitó la actividad de [name] debido a:"},{"t":"checks","items":[{"name":"q2_086","label":"Sí","rowLabel":"dolor?"},{"name":"q2_087","label":"Sí","rowLabel":"salud en general?"},{"name":"q2_088","label":"Sí","rowLabel":"instrucciones del médico o de los padres?"},{"name":"q2_089","label":"Sí","rowLabel":"miedo a no ser aceptado por los otros niños?"},{"name":"q2_090","label":"Sí","rowLabel":"que los amigos no se encontraban a su alrededor?"}]},{"t":"text","text":"Comentario Adicional para las Selecciones Anteriores:"},{"t":"textarea","name":"q2_085comment"}] } },
    { n: 21, blocks: { en: [{"t":"text","text":"Pediatric Health Assessment (parent reported)"},{"t":"text","text":"How often in the last week did [name] participate in gym/recess ?"},{"t":"radios","name":"q2_091","options":[{"value":1,"label":"Often"},{"value":2,"label":"Sometimes"},{"value":3,"label":"Never or rarely"},{"value":4,"label":"No gym or recess"}]}], sp: [{"t":"text","text":"Evaluación de Salud Pediátrica (informado por los padres)"},{"t":"text","text":"¿Con qué frecuencia durante la semana pasada participó [name] en gimnasia o en actividades deportivas?"},{"t":"radios","name":"q2_091","options":[{"value":1,"label":"Frecuentemente"},{"value":2,"label":"Algunas veces"},{"value":3,"label":"Nunca o rara vez"},{"value":4,"label":"No fue a gimnasia ni deportes"}]}] }, required: ["q2_091"] },
    { n: 22, blocks: { en: [{"t":"text","text":"Pediatric Health Assessment (parent reported)"},{"t":"text","text":"How often in the last week did [name] participate in gym/recess ? (cont.)"},{"t":"text","text":"Was [name]'s activity limited by:"},{"t":"checks","items":[{"name":"q2_092","label":"Yes","rowLabel":"Pain?"},{"name":"q2_093","label":"Yes","rowLabel":"General Health?"},{"name":"q2_094","label":"Yes","rowLabel":"Doctor or parent instructions?"},{"name":"q2_095","label":"Yes","rowLabel":"Fear the other kids won't like him/her?"},{"name":"q2_096","label":"Yes","rowLabel":"Dislike of gym/recess?"},{"name":"q2_097","label":"Yes","rowLabel":"School not in session?"},{"name":"q2_098","label":"Yes","rowLabel":"Does not attend school ?"}]},{"t":"text","text":"Additional Comment for Selection(s) Above:"},{"t":"textarea","name":"q2_091comment"}], sp: [{"t":"text","text":"Evaluación de Salud Pediátrica (informado por los padres)"},{"t":"text","text":"¿Con qué frecuencia durante la semana pasada participó [name] en gimnasia o en actividades deportivas?"},{"t":"text","text":"¿se limitó la actividad de [name] debido a:"},{"t":"checks","items":[{"name":"q2_092","label":"Sí","rowLabel":"dolor?"},{"name":"q2_093","label":"Sí","rowLabel":"salud en general?"},{"name":"q2_094","label":"Sí","rowLabel":"instrucciones del médico o de los padres?"},{"name":"q2_095","label":"Sí","rowLabel":"miedo a no ser aceptado por los otros niños?"},{"name":"q2_096","label":"Sí","rowLabel":"su disgusto por la gimnasia/deportes?"},{"name":"q2_097","label":"Sí","rowLabel":"vacaciones escolares?"},{"name":"q2_098","label":"Sí","rowLabel":"no va a la escuela?"}]},{"t":"text","text":"Additional Comment for Selection(s) Above:"},{"t":"textarea","name":"q2_091comment"}] } },
    { n: 23, blocks: { en: [{"t":"text","text":"Pediatric Health Assessment (parent reported)"},{"t":"text","text":"Is it easy or hard for [name] to make friends with children his/her own age?"},{"t":"radios","name":"q2_099","options":[{"value":1,"label":"Usually easy"},{"value":2,"label":"Sometimes easy"},{"value":3,"label":"Sometimes hard"},{"value":4,"label":"Usually hard"}]}], sp: [{"t":"text","text":"Evaluación de Salud Pediátrica (informado por los padres)"},{"t":"text","text":"¿Es fácil o difícil para [name] hacer amistad con niños de su misma edad?"},{"t":"radios","name":"q2_099","options":[{"value":1,"label":"Normalmente fácil"},{"value":2,"label":"Algunas veces fácil"},{"value":3,"label":"Algunas veces difícil"},{"value":4,"label":"Normalmente difícil"}]}] }, required: ["q2_099"] },
    { n: 24, blocks: { en: [{"t":"text","text":"Pediatric Health Assessment (parent reported)"},{"t":"text","text":"How much pain has [name] had during the last week ?"},{"t":"radios","name":"q2_100","options":[{"value":1,"label":"None"},{"value":2,"label":"Very mild"},{"value":3,"label":"Mild"},{"value":4,"label":"Moderate"},{"value":5,"label":"Severe"},{"value":6,"label":"Very severe"}]}], sp: [{"t":"text","text":"Evaluación de Salud Pediátrica (informado por los padres)"},{"t":"text","text":"¿Cuánto dolor ha tenido [name] durante la semana pasada ?"},{"t":"radios","name":"q2_100","options":[{"value":1,"label":"Ninguno"},{"value":2,"label":"Muy leve"},{"value":3,"label":"Leve"},{"value":4,"label":"Moderado"},{"value":5,"label":"Fuerte"},{"value":6,"label":"Muy fuerte"}]}] }, required: ["q2_100"] },
    { n: 25, blocks: { en: [{"t":"text","text":"Pediatric Health Assessment (parent reported)"},{"t":"text","text":"During the last week , how much did pain interfere with [name]'s normal activities (including at home, outside of the home, and at school)?"},{"t":"radios","name":"q2_101","options":[{"value":1,"label":"Not at all"},{"value":2,"label":"A little bit"},{"value":3,"label":"Moderately"},{"value":4,"label":"Quite a bit"},{"value":5,"label":"Extremely"}]}], sp: [{"t":"text","text":"Evaluación de Salud Pediátrica (informado por los padres)"},{"t":"text","text":"Durante la semana pasada , ¿hasta qué grado interfirió el dolor con las actividades normales de [name] (incluyendo en la casa, fuera de la casa y en la escuela)?"},{"t":"radios","name":"q2_101","options":[{"value":1,"label":"Para nada"},{"value":2,"label":"Un poco"},{"value":3,"label":"Moderadamente"},{"value":4,"label":"Mucho"},{"value":5,"label":"Extremadamente"}]}] }, required: ["q2_101"] },
    { n: 26, blocks: { en: [{"t":"text","text":"Pediatric Health Assessment (parent reported)"},{"t":"text","text":"What expectations do you have for [name]'s treatment?"},{"t":"text","text":"As a result of [name]'s treatment, I expect [name]:"},{"t":"scale","columns":["Definitely Yes","Probably Yes","Not Sure","Probably Not","Definitely Not"],"rows":[{"name":"q2_102","text":"To have pain relief"},{"name":"q2_103","text":"To look better"},{"name":"q2_104","text":"To feel better about himself/herself"},{"name":"q2_105","text":"To sleep more comfortably"},{"name":"q2_106","text":"To be able to do activities at home"},{"name":"q2_107","text":"To be able to do more at school"},{"name":"q2_108","text":"To be able to do more play or recreational activities (biking, walking, doing things with friends)"},{"name":"q2_109","text":"To be able to do more sports"},{"name":"q2_110","text":"To be free from pain or disability as an adult"}]}], sp: [{"t":"text","text":"Evaluación de Salud Pediátrica (informado por los padres)"},{"t":"text","text":"¿Qué expectativas tiene para el tratamiento de [name]?"},{"t":"text","text":"Como resultado del tratamiento de [name], espero que [name]:"},{"t":"scale","columns":["Definitivamente sí","Probablemente sí","No estoy seguro(a)","Probablemente no","Definitivamente no"],"rows":[{"name":"q2_102","text":"tenga alivio a su dolor."},{"name":"q2_103","text":"se vea mejor."},{"name":"q2_104","text":"se sienta mejor consigo mismo(a)."},{"name":"q2_105","text":"duerma mejor y más cómodamente."},{"name":"q2_106","text":"pueda hacer más actividades en casa."},{"name":"q2_107","text":"pueda hacer más en la escuela."},{"name":"q2_108","text":"pueda jugar más o realizar más actividades recreativas (andar en bicicleta, caminar, hacer cosas con amigos)."},{"name":"q2_109","text":"pueda practicar más deportes."},{"name":"q2_110","text":"esté libre de dolor o incapacidades como adulto."}]}] }, required: ["q2_102","q2_103","q2_104","q2_105","q2_106","q2_107","q2_108","q2_109","q2_110"] },
    { n: 27, blocks: { en: [{"t":"text","text":"Pediatric Health Assessment (parent reported)"},{"t":"text","text":"If [name] had to spend the rest of his/her life with his/her bone and muscle condition as it is right now , how would you feel about it?"},{"t":"radios","name":"q2_111","options":[{"value":1,"label":"Very satisfied"},{"value":2,"label":"Somewhat satisfied"},{"value":3,"label":"Neutral"},{"value":4,"label":"Somewhat dissatisfied"},{"value":5,"label":"Very dissatisfied"}]}], sp: [{"t":"text","text":"Evaluación de Salud Pediátrica (informado por los padres)"},{"t":"text","text":"Si [name] tuviera que pasar el resto de su vida con su condición de los huesos y músculos como la que tiene ahora , ¿cómo se sentiría al respecto?"},{"t":"radios","name":"q2_111","options":[{"value":1,"label":"Muy satisfecho(a)"},{"value":2,"label":"Algo satisfecho(a)"},{"value":3,"label":"Neutral"},{"value":4,"label":"Algo insa-tisfecho(a)"},{"value":5,"label":"Muy insa-tisfecho(a)"}]}] }, required: ["q2_111"] },
  ],
  AP: [
    { n: 0, blocks: { en: [{"t":"text","text":"The following is a survey designed by a group of doctors [1] to measure the quality of life of the children they treat."},{"t":"text","text":"Your answers will help us to understand how your child's condition affects his or her daily functions and overall happiness."},{"t":"text","text":"It will also let us see if the treatment your child receives improves his or her quality of life."},{"t":"text","text":"We appreciate your input and thank you for allowing us to provide this service."},{"t":"text","text":"[1] The Pediatric Outcomes Data Collection Instrument (PODCI), Version 2.0, revised Aug 2005."}] } },
    { n: 1, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (parent reported)"},{"t":"text","text":"In general, would you say [name]'s health is:"},{"t":"radios","name":"q2_005","options":[{"value":1,"label":"Excellent"},{"value":2,"label":"Very Good"},{"value":3,"label":"Good"},{"value":4,"label":"Fair"},{"value":5,"label":"Poor"}]}] } },
    { n: 2, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (parent reported)"},{"t":"text","text":"Compared to one year ago, how would you rate [name]'s health in general:"},{"t":"radios","name":"q2_006","options":[{"value":1,"label":"Much better now than one year ago"},{"value":2,"label":"Somewhat better now than one year ago"},{"value":3,"label":"About the same as one year ago"},{"value":4,"label":"Somewhat worse now than one year ago"},{"value":5,"label":"Much worse now than one year ago"}]}] } },
    { n: 3, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (parent reported)"},{"t":"text","text":"Have you ever been told by a doctor, nurse, teacher, or other health professional that [name] has had any of the following conditions?"},{"t":"text","text":"If yes, indicate if your child is being treated for this condition and if your child is limited by those conditions."},{"t":"grid","columns":["Has your child ever had it?","Does your child receive treatment for it now?","Are your child's activities limited by it now?"],"rows":[{"label":"Juvenile Arthritis (one or two joints)","cells":[{"name":"q2_007a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_007b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_007c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Juvenile Arthritis (many joints)","cells":[{"name":"q2_008a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_008b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_008c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Anorexia or bulemia (eating disorders)","cells":[{"name":"q2_009a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_009b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_009c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Asthma","cells":[{"name":"q2_010a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_010b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_010c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Attention or behavioral problems","cells":[{"name":"q2_011a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_011b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_011c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Chronic allergies or sinus trouble","cells":[{"name":"q2_012a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_012b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_012c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Developmental delay","cells":[{"name":"q2_013a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_013b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_013c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Cognitive Impairment","cells":[{"name":"q2_014a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_014b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_014c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Diabetes","cells":[{"name":"q2_015a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_015b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_015c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Epilepsy (seizure disorder)","cells":[{"name":"q2_016a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_016b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_016c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Hearing impairment or deafness","cells":[{"name":"q2_017a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_017b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_017c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Heart problem","cells":[{"name":"q2_018a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_018b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_018c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Learning problem","cells":[{"name":"q2_019a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_019b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_019c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Sleep disturbance","cells":[{"name":"q2_020a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_020b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_020c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Speech problems","cells":[{"name":"q2_021a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_021b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_021c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Vision problems","cells":[{"name":"q2_022a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_022b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_022c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]}]}] }, required: ["q2_007a","q2_008a","q2_009a","q2_010a","q2_011a","q2_012a","q2_013a","q2_014a","q2_015a","q2_016a","q2_017a","q2_018a","q2_019a","q2_020a","q2_021a","q2_022a","q2_007b","q2_008b","q2_009b","q2_010b","q2_011b","q2_012b","q2_013b","q2_014b","q2_015b","q2_016b","q2_017b","q2_018b","q2_019b","q2_020b","q2_021b","q2_022b","q2_007c","q2_008c","q2_009c","q2_010c","q2_011c","q2_012c","q2_013c","q2_014c","q2_015c","q2_016c","q2_017c","q2_018c","q2_019c","q2_020c","q2_021c","q2_022c"] },
    { n: 4, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (parent reported)"},{"t":"text","text":"For [name]'s front right side , please indicate those areas that bother him/her or limit his/her function."},{"t":"checks","items":[{"name":"q2_023a","label":"Neck","rowLabel":""},{"name":"q2_023b","label":"Shoulder area","rowLabel":""},{"name":"q2_023c","label":"Elbow/Forearm","rowLabel":""},{"name":"q2_023d","label":"Wrist/Hand","rowLabel":""},{"name":"q2_023e","label":"Hip","rowLabel":""},{"name":"q2_023f","label":"Thigh","rowLabel":""},{"name":"q2_023g","label":"Knee area","rowLabel":""},{"name":"q2_023h","label":"Calf area","rowLabel":""},{"name":"q2_023i","label":"Ankle/Foot area","rowLabel":""}]},{"t":"text","text":"For [name]'s front left side , please indicate those areas that bother him/her or limit his/her function."},{"t":"checks","items":[{"name":"q2_024a","label":"Neck","rowLabel":""},{"name":"q2_024b","label":"Shoulder area","rowLabel":""},{"name":"q2_024c","label":"Elbow/Forearm","rowLabel":""},{"name":"q2_024d","label":"Wrist/Hand","rowLabel":""},{"name":"q2_024e","label":"Hip","rowLabel":""},{"name":"q2_024f","label":"Thigh","rowLabel":""},{"name":"q2_024g","label":"Knee area","rowLabel":""},{"name":"q2_024h","label":"Calf area","rowLabel":""},{"name":"q2_024i","label":"Ankle/Foot area","rowLabel":""}]},{"t":"text","text":"For [name]'s back , please indicate those areas that bother him/her or limit his/her function."},{"t":"checks","items":[{"name":"q2_025a","label":"Neck","rowLabel":""},{"name":"q2_025b","label":"Upper Back","rowLabel":""},{"name":"q2_025c","label":"Lower Back","rowLabel":""}]}] } },
    { n: 5, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (parent reported)"},{"t":"text","text":"Some kinds of problems can make it hard to do many activities, such as eating, bathing, school work, and playing with friends."},{"t":"text","text":"We would like to find out how [name] is doing."},{"t":"text","text":"During the last week was it easy or hard for your child to:"},{"t":"scale","columns":["Easy","A little hard","Very hard","Can't do at all","Too young for this activity"],"rows":[{"name":"q2_026","text":"Lift heavy books?"},{"name":"q2_027","text":"Pour a half gallon of milk?"},{"name":"q2_028","text":"Open a jar that has been opened before?"},{"name":"q2_029","text":"Use a fork and spoon?"},{"name":"q2_030","text":"Comb his/her hair?"},{"name":"q2_031","text":"Button buttons?"},{"name":"q2_032x","text":"Put on his/her coat?"},{"name":"q2_033","text":"Write with a pencil?"}]}] } },
    { n: 6, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (parent reported)"},{"t":"text","text":"On average, over the last 12 months , how often did [name] miss school (camp, etc.) because of his/her health?"},{"t":"radios","name":"q2_034","options":[{"value":1,"label":"Rarely"},{"value":2,"label":"Once a month"},{"value":3,"label":"Two or three times a month"},{"value":4,"label":"Once a week"},{"value":5,"label":"More than once a week"},{"value":6,"label":"Does not attend school, etc."}]}] }, required: ["q2_034"] },
    { n: 7, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (parent reported)"},{"t":"text","text":"During the last week , how happy has [name] been with:"},{"t":"scale","columns":["Very happy","Somewhat happy","Not sure","Somewhat unhappy","Very unhappy","Child is too young"],"rows":[{"name":"q2_035","text":"How he/she looks?"},{"name":"q2_036","text":"His/her body?"},{"name":"q2_037","text":"What clothes or shoes he/she can wear"},{"name":"q2_038","text":"His/her ability to do the same things his/her friends do?"},{"name":"q2_039","text":"His/her health in general?"}]}] }, required: ["q2_035","q2_036","q2_037","q2_038","q2_039"] },
    { n: 8, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (parent reported)"},{"t":"text","text":"During the last week , how much of the time:"},{"t":"scale","columns":["Most of the time","Some of the time","A little of the time","None of the time"],"rows":[{"name":"q2_040","text":"Did [name] feel sick and tired?"},{"name":"q2_041","text":"Was [name] full of pep and energy?"},{"name":"q2_042","text":"Did pain or discomfort interfere with [name]'s activities?"}]}] }, required: ["q2_040","q2_041","q2_042"] },
    { n: 9, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (parent reported)"},{"t":"text","text":"During the last week , has it been easy or hard for [name] to:"},{"t":"scale","columns":["Easy","A little hard","Very hard","Can't do at all","Too young for this activity"],"rows":[{"name":"q2_043","text":"Run short distances?"},{"name":"q2_044","text":"Bicycle or tricycle?"},{"name":"q2_045","text":"Climb three flights of stairs?"},{"name":"q2_046","text":"Climb one flight of stairs?"},{"name":"q2_047","text":"Walk more than a mile?"},{"name":"q2_048","text":"Walk three blocks?"},{"name":"q2_049","text":"Walk one block?"},{"name":"q2_050","text":"Get on and off a bus?"}]}] }, required: ["q2_043","q2_044","q2_045","q2_046","q2_047","q2_048","q2_049","q2_050"] },
    { n: 10, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (parent reported)"},{"t":"text","text":"How often does [name] need help from another person for walking and climbing?"},{"t":"radios","name":"q2_051","options":[{"value":1,"label":"Never"},{"value":2,"label":"Sometimes"},{"value":3,"label":"About half the time"},{"value":4,"label":"Often"},{"value":5,"label":"All the time"}]},{"t":"text","text":"How often does [name] use assistive devices (such as braces, crutches, or wheelchair) for walking and climbing?"},{"t":"radios","name":"q2_052","options":[{"value":1,"label":"Never"},{"value":2,"label":"Sometimes"},{"value":3,"label":"About half the time"},{"value":4,"label":"Often"},{"value":5,"label":"All the time"}]}] }, required: ["q2_051","q2_052"] },
    { n: 11, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (parent reported)"},{"t":"text","text":"During the last week , has it been easy or hard for [name] to:"},{"t":"scale","columns":["Easy","A little hard","Very hard","Can't do at all","Too young for this activity"],"rows":[{"name":"q2_053","text":"Stand while washing his/her hands and face at a sink?"},{"name":"q2_054","text":"Sit in a regular chair without holding on?"},{"name":"q2_055","text":"Get on and off a toilet or chair?"},{"name":"q2_056","text":"Get in and out of bed?"},{"name":"q2_057","text":"Turn door knobs?"},{"name":"q2_058","text":"Bend over from a standing position and pick up something off the floor?"}]}] } },
    { n: 12, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (parent reported)"},{"t":"text","text":"How often does [name] need help from another person for sitting and standing?"},{"t":"radios","name":"q2_059","options":[{"value":1,"label":"Never"},{"value":2,"label":"Sometimes"},{"value":3,"label":"About half the time"},{"value":4,"label":"Often"},{"value":5,"label":"All the time"}]},{"t":"text","text":"How often does [name] use assistive devices (such as braces, crutches, or wheelchair) for sitting and standing?"},{"t":"radios","name":"q2_060","options":[{"value":1,"label":"Never"},{"value":2,"label":"Sometimes"},{"value":3,"label":"About half the time"},{"value":4,"label":"Often"},{"value":5,"label":"All the time"}]}] }, required: ["q2_059","q2_060"] },
    { n: 13, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (parent reported)"},{"t":"text","text":"Can [name] participate in recreational outdoor activities with other children the same age?"},{"t":"text","text":"(For example: bicycling, skating, hiking, jogging)"},{"t":"radios","name":"q2_061","options":[{"value":1,"label":"Yes, easily"},{"value":2,"label":"Yes, but a little hard"},{"value":3,"label":"Yes, but very hard"},{"value":4,"label":"No"}]}] }, required: ["q2_061"] },
    { n: 14, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (parent reported)"},{"t":"text","text":"Can [name] participate in recreational outdoor activities with other children the same age? (cont.)"},{"t":"text","text":"(For example: bicycling, skating, hiking, jogging)"},{"t":"text","text":"Was [name]'s activity limited by:"},{"t":"checks","items":[{"name":"q2_062","label":"Yes","rowLabel":"Pain?"},{"name":"q2_063","label":"Yes","rowLabel":"General Health?"},{"name":"q2_064","label":"Yes","rowLabel":"Doctor or parent instructions?"},{"name":"q2_065","label":"Yes","rowLabel":"Fear the other kids won't like him/her?"},{"name":"q2_066","label":"Yes","rowLabel":"Dislike of recreational outdoor activities?"},{"name":"q2_067","label":"Yes","rowLabel":"Too young?"},{"name":"q2_068","label":"Yes","rowLabel":"Activity not in season?"}]},{"t":"text","text":"Additional Comment for Selection(s) Above:"},{"t":"textarea","name":"q2_061comment"}] } },
    { n: 15, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (parent reported)"},{"t":"text","text":"Can [name] participate in pickup games or sports with other children the same age?"},{"t":"text","text":"(For example: tag, dodge ball, basketball, softball, soccer, catch, jump rope, touch football, hop scotch)"},{"t":"radios","name":"q2_069","options":[{"value":1,"label":"Yes, easily"},{"value":2,"label":"Yes, but a little hard"},{"value":3,"label":"Yes, but very hard"},{"value":4,"label":"No"}]}] }, required: ["q2_069"] },
    { n: 16, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (parent reported)"},{"t":"text","text":"Can [name] participate in pickup games or sports with other children the same age? (cont.)"},{"t":"text","text":"(For example: tag, dodge ball, basketball, softball, soccer, catch, jump rope, touch football, hop scotch)"},{"t":"text","text":"Was [name]'s activity limited by:"},{"t":"checks","items":[{"name":"q2_070","label":"Yes","rowLabel":"Pain?"},{"name":"q2_071","label":"Yes","rowLabel":"General Health?"},{"name":"q2_072","label":"Yes","rowLabel":"Doctor or parent instructions?"},{"name":"q2_073","label":"Yes","rowLabel":"Fear the other kids won't like him/her?"},{"name":"q2_074","label":"Yes","rowLabel":"Dislike of pickup games or sports?"},{"name":"q2_075","label":"Yes","rowLabel":"Too young?"},{"name":"q2_076","label":"Yes","rowLabel":"Activity not in season?"}]},{"t":"text","text":"Additional Comment for Selection(s) Above:"},{"t":"textarea","name":"q2_069comment"}] } },
    { n: 17, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (parent reported)"},{"t":"text","text":"Can [name] participate in competitive level sports with other children the same age?"},{"t":"text","text":"(For example: hockey, basketball, soccer, football, baseball, swimming, running [track or cross country], gymnastics, or dance)"},{"t":"radios","name":"q2_077","options":[{"value":1,"label":"Yes, easily"},{"value":2,"label":"Yes, but a little hard"},{"value":3,"label":"Yes, but very hard"},{"value":4,"label":"No"}]}] }, required: ["q2_077"] },
    { n: 18, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (parent reported)"},{"t":"text","text":"Can [name] participate in competitive level sports with other children the same age? (cont.)"},{"t":"text","text":"(For example: hockey, basketball, soccer, football, baseball, swimming, running [track or cross country], gymnastics, or dance)"},{"t":"text","text":"Was [name]'s activity limited by:"},{"t":"checks","items":[{"name":"q2_078","label":"Yes","rowLabel":"Pain?"},{"name":"q2_079","label":"Yes","rowLabel":"General Health?"},{"name":"q2_080","label":"Yes","rowLabel":"Doctor or parent instructions?"},{"name":"q2_081","label":"Yes","rowLabel":"Fear the other kids won't like him/her?"},{"name":"q2_082","label":"Yes","rowLabel":"Dislike of competitive level sports?"},{"name":"q2_083","label":"Yes","rowLabel":"Too young?"},{"name":"q2_084","label":"Yes","rowLabel":"Activity not in season?"}]},{"t":"text","text":"Additional Comment for Selection(s) Above:"},{"t":"textarea","name":"q2_077comment"}] } },
    { n: 19, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (parent reported)"},{"t":"text","text":"How often in the last week did [name] get together and do things with friends?"},{"t":"radios","name":"q2_085","options":[{"value":1,"label":"Often"},{"value":2,"label":"Sometimes"},{"value":3,"label":"Never or rarely"}]}] }, required: ["q2_085"] },
    { n: 20, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (parent reported)"},{"t":"text","text":"How often in the last week did [name] get together and do things with friends? (cont.)"},{"t":"text","text":"Was [name]'s activity limited by:"},{"t":"checks","items":[{"name":"q2_086","label":"Yes","rowLabel":"Pain?"},{"name":"q2_087","label":"Yes","rowLabel":"General Health?"},{"name":"q2_088","label":"Yes","rowLabel":"Doctor or parent instructions?"},{"name":"q2_089","label":"Yes","rowLabel":"Fear the other kids won't like him/her?"},{"name":"q2_090","label":"Yes","rowLabel":"Friends not around?"}]},{"t":"text","text":"Additional Comment for Selection(s) Above:"},{"t":"textarea","name":"q2_085comment"}] } },
    { n: 21, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (parent reported)"},{"t":"text","text":"How often in the last week did [name] participate in gym/recess ?"},{"t":"radios","name":"q2_091","options":[{"value":1,"label":"Often"},{"value":2,"label":"Sometimes"},{"value":3,"label":"Never or rarely"},{"value":4,"label":"No gym or recess"}]}] }, required: ["q2_091"] },
    { n: 22, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (parent reported)"},{"t":"text","text":"How often in the last week did [name] participate in gym/recess ? (cont.)"},{"t":"text","text":"Was [name]'s activity limited by:"},{"t":"checks","items":[{"name":"q2_092","label":"Yes","rowLabel":"Pain?"},{"name":"q2_093","label":"Yes","rowLabel":"General Health?"},{"name":"q2_094","label":"Yes","rowLabel":"Doctor or parent instructions?"},{"name":"q2_095","label":"Yes","rowLabel":"Fear the other kids won't like him/her?"},{"name":"q2_096","label":"Yes","rowLabel":"Dislike of gym/recess?"},{"name":"q2_097","label":"Yes","rowLabel":"School not in session?"},{"name":"q2_098","label":"Yes","rowLabel":"Does not attend school ?"}]},{"t":"text","text":"Additional Comment for Selection(s) Above:"},{"t":"textarea","name":"q2_091comment"}] } },
    { n: 23, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (parent reported)"},{"t":"text","text":"Is it easy or hard for [name] to make friends with children his/her own age?"},{"t":"radios","name":"q2_099","options":[{"value":1,"label":"Usually easy"},{"value":2,"label":"Sometimes easy"},{"value":3,"label":"Sometimes hard"},{"value":4,"label":"Usually hard"}]}] }, required: ["q2_099"] },
    { n: 24, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (parent reported)"},{"t":"text","text":"How much pain has [name] had during the last week ?"},{"t":"radios","name":"q2_100","options":[{"value":1,"label":"None"},{"value":2,"label":"Very mild"},{"value":3,"label":"Mild"},{"value":4,"label":"Moderate"},{"value":5,"label":"Severe"},{"value":6,"label":"Very severe"}]}] }, required: ["q2_100"] },
    { n: 25, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (parent reported)"},{"t":"text","text":"During the last week , how much did pain interfere with [name]'s normal activities (including at home, outside of the home, and at school)?"},{"t":"radios","name":"q2_101","options":[{"value":1,"label":"Not at all"},{"value":2,"label":"A little bit"},{"value":3,"label":"Moderately"},{"value":4,"label":"Quite a bit"},{"value":5,"label":"Extremely"}]}] }, required: ["q2_101"] },
    { n: 26, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (parent reported)"},{"t":"text","text":"What expectations do you have for [name]'s treatment?"},{"t":"text","text":"As a result of [name]'s treatment, I expect [name]:"},{"t":"scale","columns":["Definitely Yes","Probably Yes","Not Sure","Probably Not","Definitely Not"],"rows":[{"name":"q2_102","text":"To have pain relief"},{"name":"q2_103","text":"To look better"},{"name":"q2_104","text":"To feel better about himself/herself"},{"name":"q2_105","text":"To sleep more comfortably"},{"name":"q2_106","text":"To be able to do activities at home"},{"name":"q2_107","text":"To be able to do more at school"},{"name":"q2_108","text":"To be able to do more play or recreational activities (biking, walking, doing things with friends)"},{"name":"q2_109","text":"To be able to do more sports"},{"name":"q2_110","text":"To be free from pain or disability as an adult"}]}] }, required: ["q2_102","q2_103","q2_104","q2_105","q2_106","q2_107","q2_108","q2_109","q2_110"] },
    { n: 27, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (parent reported)"},{"t":"text","text":"If [name] had to spend the rest of his/her life with his/her bone and muscle condition as it is right now , how would you feel about it?"},{"t":"radios","name":"q2_111","options":[{"value":1,"label":"Very satisfied"},{"value":2,"label":"Somewhat satisfied"},{"value":3,"label":"Neutral"},{"value":4,"label":"Somewhat dissatisfied"},{"value":5,"label":"Very dissatisfied"}]}] }, required: ["q2_111"] },
  ],
  AS: [
    { n: 0, blocks: { en: [{"t":"text","text":"The following is a survey designed by a group of doctors [1] to measure the quality of life they treat."},{"t":"text","text":"Your answers will help us to understand how your condition affects your daily functions and overall happiness."},{"t":"text","text":"It will also let us see if the treatment you receive improves your quality of life."},{"t":"text","text":"We appreciate your input and thank you for allowing us to provide this service."},{"t":"text","text":"[1] The Pediatric Outcomes Data Collection Instrument (PODCI), Version 2.0, revised Aug 2005."}] } },
    { n: 1, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (self reported)"},{"t":"text","text":"In general, would you say your health is:"},{"t":"radios","name":"q2_005","options":[{"value":1,"label":"Excellent"},{"value":2,"label":"Very Good"},{"value":3,"label":"Good"},{"value":4,"label":"Fair"},{"value":5,"label":"Poor"}]}] } },
    { n: 2, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (self reported)"},{"t":"text","text":"Compared to one year ago, how would you rate your health in general:"},{"t":"radios","name":"q2_006","options":[{"value":1,"label":"Much better now than one year ago"},{"value":2,"label":"Somewhat better now than one year ago"},{"value":3,"label":"About the same as one year ago"},{"value":4,"label":"Somewhat worse now than one year ago"},{"value":5,"label":"Much worse now than one year ago"}]}] } },
    { n: 3, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (self reported)"},{"t":"text","text":"Have you ever been told by a doctor, nurse, teacher, or other health professional that you have had any of the following conditions?"},{"t":"text","text":"If yes, indicate if you are being treated for this condition and if you are limited by those conditions."},{"t":"grid","columns":["Have you ever had it?","Do you receive treatment for it now?","Are your activities limited by it now?"],"rows":[{"label":"Juvenile Arthritis (one or two joints)","cells":[{"name":"q2_007a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_007b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_007c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Juvenile Arthritis (many joints)","cells":[{"name":"q2_008a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_008b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_008c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Anorexia or bulemia (eating disorders)","cells":[{"name":"q2_009a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_009b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_009c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Asthma","cells":[{"name":"q2_010a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_010b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_010c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Attention or behavioral problems","cells":[{"name":"q2_011a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_011b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_011c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Chronic allergies or sinus trouble","cells":[{"name":"q2_012a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_012b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_012c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Developmental delay","cells":[{"name":"q2_013a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_013b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_013c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Cognitive Impairment","cells":[{"name":"q2_014a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_014b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_014c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Diabetes","cells":[{"name":"q2_015a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_015b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_015c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Epilepsy (seizure disorder)","cells":[{"name":"q2_016a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_016b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_016c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Hearing impairment or deafness","cells":[{"name":"q2_017a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_017b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_017c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Heart problem","cells":[{"name":"q2_018a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_018b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_018c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Learning problem","cells":[{"name":"q2_019a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_019b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_019c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Sleep disturbance","cells":[{"name":"q2_020a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_020b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_020c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Speech problems","cells":[{"name":"q2_021a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_021b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_021c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]},{"label":"Vision problems","cells":[{"name":"q2_022a","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_022b","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]},{"name":"q2_022c","options":[{"value":1,"label":"Yes"},{"value":2,"label":"No"}]}]}]}] }, required: ["q2_007a","q2_008a","q2_009a","q2_010a","q2_011a","q2_012a","q2_013a","q2_014a","q2_015a","q2_016a","q2_017a","q2_018a","q2_019a","q2_020a","q2_021a","q2_022a","q2_007b","q2_008b","q2_009b","q2_010b","q2_011b","q2_012b","q2_013b","q2_014b","q2_015b","q2_016b","q2_017b","q2_018b","q2_019b","q2_020b","q2_021b","q2_022b","q2_007c","q2_008c","q2_009c","q2_010c","q2_011c","q2_012c","q2_013c","q2_014c","q2_015c","q2_016c","q2_017c","q2_018c","q2_019c","q2_020c","q2_021c","q2_022c"] },
    { n: 4, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (self reported)"},{"t":"text","text":"For your front right side , please indicate those areas that bother you or limit your function."},{"t":"checks","items":[{"name":"q2_023a","label":"Neck","rowLabel":""},{"name":"q2_023b","label":"Shoulder area","rowLabel":""},{"name":"q2_023c","label":"Elbow/Forearm","rowLabel":""},{"name":"q2_023d","label":"Wrist/Hand","rowLabel":""},{"name":"q2_023e","label":"Hip","rowLabel":""},{"name":"q2_023f","label":"Thigh","rowLabel":""},{"name":"q2_023g","label":"Knee area","rowLabel":""},{"name":"q2_023h","label":"Calf area","rowLabel":""},{"name":"q2_023i","label":"Ankle/Foot area","rowLabel":""}]},{"t":"text","text":"For your front left side , please indicate those areas that bother you or limit your function."},{"t":"checks","items":[{"name":"q2_024a","label":"Neck","rowLabel":""},{"name":"q2_024b","label":"Shoulder area","rowLabel":""},{"name":"q2_024c","label":"Elbow/Forearm","rowLabel":""},{"name":"q2_024d","label":"Wrist/Hand","rowLabel":""},{"name":"q2_024e","label":"Hip","rowLabel":""},{"name":"q2_024f","label":"Thigh","rowLabel":""},{"name":"q2_024g","label":"Knee area","rowLabel":""},{"name":"q2_024h","label":"Calf area","rowLabel":""},{"name":"q2_024i","label":"Ankle/Foot area","rowLabel":""}]},{"t":"text","text":"For your back , please indicate those areas that bother you or limit your function."},{"t":"checks","items":[{"name":"q2_025a","label":"Neck","rowLabel":""},{"name":"q2_025b","label":"Upper Back","rowLabel":""},{"name":"q2_025c","label":"Lower Back","rowLabel":""}]}] } },
    { n: 5, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (self reported)"},{"t":"text","text":"Some kinds of problems can make it hard to do many activities, such as eating, bathing, school work, and playing with friends."},{"t":"text","text":"We would like to find out how you are doing."},{"t":"text","text":"During the last week was it easy or hard for you to:"},{"t":"scale","columns":["Easy","A little hard","Very hard","Can't do at all"],"rows":[{"name":"q2_026","text":"Lift heavy books?"},{"name":"q2_027","text":"Pour a half gallon of milk?"},{"name":"q2_028","text":"Open a jar that has been opened before?"},{"name":"q2_029","text":"Use a fork and spoon?"},{"name":"q2_030","text":"Comb your hair?"},{"name":"q2_031","text":"Button buttons?"},{"name":"q2_032x","text":"Put on your coat?"},{"name":"q2_033","text":"Write with a pencil?"}]}] } },
    { n: 6, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (self reported)"},{"t":"text","text":"On average, over the last 12 months , how often did you miss school (camp, etc.) because of your health?"},{"t":"radios","name":"q2_034","options":[{"value":1,"label":"Rarely"},{"value":2,"label":"Once a month"},{"value":3,"label":"Two or three times a month"},{"value":4,"label":"Once a week"},{"value":5,"label":"More than once a week"},{"value":6,"label":"Do not attend school, etc."}]}] }, required: ["q2_034"] },
    { n: 7, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (self reported)"},{"t":"text","text":"During the last week , how happy have you been with:"},{"t":"scale","columns":["Very happy","Somewhat happy","Not sure","Somewhat unhappy","Very unhappy"],"rows":[{"name":"q2_035","text":"How you look?"},{"name":"q2_036","text":"Your body?"},{"name":"q2_037","text":"What clothes or shoes you can wear"},{"name":"q2_038","text":"Your ability to do the same things your friends do?"},{"name":"q2_039","text":"Your health in general?"}]}] }, required: ["q2_035","q2_036","q2_037","q2_038","q2_039"] },
    { n: 8, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (self reported)"},{"t":"text","text":"During the last week , how much of the time:"},{"t":"scale","columns":["Most of the time","Some of the time","A little of the time","None of the time"],"rows":[{"name":"q2_040","text":"Did you feel sick and tired?"},{"name":"q2_041","text":"Were you full of pep and energy?"},{"name":"q2_042","text":"Did pain or discomfort interfere with your activities?"}]}] }, required: ["q2_040","q2_041","q2_042"] },
    { n: 9, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (self reported)"},{"t":"text","text":"During the last week , has it been easy or hard for you to:"},{"t":"scale","columns":["Easy","A little hard","Very hard","Can't do at all"],"rows":[{"name":"q2_043","text":"Run short distances?"},{"name":"q2_044","text":"Bicycle or tricycle?"},{"name":"q2_045","text":"Climb three flights of stairs?"},{"name":"q2_046","text":"Climb one flight of stairs?"},{"name":"q2_047","text":"Walk more than a mile?"},{"name":"q2_048","text":"Walk three blocks?"},{"name":"q2_049","text":"Walk one block?"},{"name":"q2_050","text":"Get on and off a bus?"}]}] }, required: ["q2_043","q2_044","q2_045","q2_046","q2_047","q2_048","q2_049","q2_050"] },
    { n: 10, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (self reported)"},{"t":"text","text":"How often do you need help from another person for walking and climbing?"},{"t":"radios","name":"q2_051","options":[{"value":1,"label":"Never"},{"value":2,"label":"Sometimes"},{"value":3,"label":"About half the time"},{"value":4,"label":"Often"},{"value":5,"label":"All the time"}]},{"t":"text","text":"How often do you use assistive devices (such as braces, crutches, or wheelchair) for walking and climbing?"},{"t":"radios","name":"q2_052","options":[{"value":1,"label":"Never"},{"value":2,"label":"Sometimes"},{"value":3,"label":"About half the time"},{"value":4,"label":"Often"},{"value":5,"label":"All the time"}]}] }, required: ["q2_051","q2_052"] },
    { n: 11, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (self reported)"},{"t":"text","text":"During the last week , has it been easy or hard for you to:"},{"t":"scale","columns":["Easy","A little hard","Very hard","Can't do at all"],"rows":[{"name":"q2_053","text":"Stand while washing your hands and face at a sink?"},{"name":"q2_054","text":"Sit in a regular chair without holding on?"},{"name":"q2_055","text":"Get on and off a toilet or chair?"},{"name":"q2_056","text":"Get in and out of bed?"},{"name":"q2_057","text":"Turn door knobs?"},{"name":"q2_058","text":"Bend over from a standing position and pick up something off the floor?"}]}] }, required: ["q2_053","q2_054","q2_055","q2_056","q2_057","q2_058"] },
    { n: 12, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (self reported)"},{"t":"text","text":"How often do you need help from another person for sitting and standing?"},{"t":"radios","name":"q2_059","options":[{"value":1,"label":"Never"},{"value":2,"label":"Sometimes"},{"value":3,"label":"About half the time"},{"value":4,"label":"Often"},{"value":5,"label":"All the time"}]},{"t":"text","text":"How often do you use assistive devices (such as braces, crutches, or wheelchair) for sitting and standing?"},{"t":"radios","name":"q2_060","options":[{"value":1,"label":"Never"},{"value":2,"label":"Sometimes"},{"value":3,"label":"About half the time"},{"value":4,"label":"Often"},{"value":5,"label":"All the time"}]}] }, required: ["q2_059","q2_060"] },
    { n: 13, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (self reported)"},{"t":"text","text":"Can you participate in recreational outdoor activities with other kids your own age?"},{"t":"text","text":"(For example: bicycling, skating, hiking, jogging)"},{"t":"radios","name":"q2_061","options":[{"value":1,"label":"Yes, easily"},{"value":2,"label":"Yes, but a little hard"},{"value":3,"label":"Yes, but very hard"},{"value":4,"label":"No"}]}] }, required: ["q2_061"] },
    { n: 14, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (self reported)"},{"t":"text","text":"Can you participate in recreational outdoor activities with other kids your own age? (cont.)"},{"t":"text","text":"(For example: bicycling, skating, hiking, jogging)"},{"t":"text","text":"Was your activity limited by:"},{"t":"checks","items":[{"name":"q2_062","label":"Yes","rowLabel":"Pain?"},{"name":"q2_063","label":"Yes","rowLabel":"General Health?"},{"name":"q2_064","label":"Yes","rowLabel":"Doctor or parent instructions?"},{"name":"q2_065","label":"Yes","rowLabel":"Fear the other kids won't like you?"},{"name":"q2_066","label":"Yes","rowLabel":"Dislike of recreational outdoor activities?"},{"name":"q2_068","label":"Yes","rowLabel":"Activity not in season?"}]},{"t":"text","text":"Additional Comment for Selection(s) Above:"},{"t":"textarea","name":"q2_061comment"}] } },
    { n: 15, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (self reported)"},{"t":"text","text":"Can you participate in pickup games or sports with other kids your own age?"},{"t":"text","text":"(For example: tag, dodge ball, basketball, softball, soccer, catch, jump rope, touch football, hop scotch)"},{"t":"radios","name":"q2_069","options":[{"value":1,"label":"Yes, easily"},{"value":2,"label":"Yes, but a little hard"},{"value":3,"label":"Yes, but very hard"},{"value":4,"label":"No"}]}] }, required: ["q2_069"] },
    { n: 16, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (self reported)"},{"t":"text","text":"Can you participate in pickup games or sports with other kids your own age? (cont.)"},{"t":"text","text":"(For example: tag, dodge ball, basketball, softball, soccer, catch, jump rope, touch football, hop scotch)"},{"t":"text","text":"Was your activity limited by:"},{"t":"checks","items":[{"name":"q2_070","label":"Yes","rowLabel":"Pain?"},{"name":"q2_071","label":"Yes","rowLabel":"General Health?"},{"name":"q2_072","label":"Yes","rowLabel":"Doctor or parent instructions?"},{"name":"q2_073","label":"Yes","rowLabel":"Fear the other kids won't like you?"},{"name":"q2_074","label":"Yes","rowLabel":"Dislike of pickup games or sports?"},{"name":"q2_076","label":"Yes","rowLabel":"Activity not in season?"}]},{"t":"text","text":"Additional Comment for Selection(s) Above:"},{"t":"textarea","name":"q2_069comment"}] } },
    { n: 17, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (self reported)"},{"t":"text","text":"Can you participate in competitive level sports with other kids your own age?"},{"t":"text","text":"(For example: hockey, basketball, soccer, football, baseball, swimming, running [track or cross country], gymnastics, or dance)"},{"t":"radios","name":"q2_077","options":[{"value":1,"label":"Yes, easily"},{"value":2,"label":"Yes, but a little hard"},{"value":3,"label":"Yes, but very hard"},{"value":4,"label":"No"}]}] }, required: ["q2_077"] },
    { n: 18, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (self reported)"},{"t":"text","text":"Can you participate in competitive level sports with other kids your own age? (cont.)"},{"t":"text","text":"(For example: hockey, basketball, soccer, football, baseball, swimming, running [track or cross country], gymnastics, or dance)"},{"t":"text","text":"Was your activity limited by:"},{"t":"checks","items":[{"name":"q2_078","label":"Yes","rowLabel":"Pain?"},{"name":"q2_079","label":"Yes","rowLabel":"General Health?"},{"name":"q2_080","label":"Yes","rowLabel":"Doctor or parent instructions?"},{"name":"q2_081","label":"Yes","rowLabel":"Fear the other kids won't like you?"},{"name":"q2_082","label":"Yes","rowLabel":"Dislike of competitive level sports?"},{"name":"q2_084","label":"Yes","rowLabel":"Activity not in season?"}]},{"t":"text","text":"Additional Comment for Selection(s) Above:"},{"t":"textarea","name":"q2_077comment"}] } },
    { n: 19, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (self reported)"},{"t":"text","text":"How often in the last week did you get together and do things with friends?"},{"t":"radios","name":"q2_085","options":[{"value":1,"label":"Often"},{"value":2,"label":"Sometimes"},{"value":3,"label":"Never or rarely"}]}] }, required: ["q2_085"] },
    { n: 20, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (self reported)"},{"t":"text","text":"How often in the last week did you get together and do things with friends? (cont.)"},{"t":"text","text":"Was your activity limited by:"},{"t":"checks","items":[{"name":"q2_086","label":"Yes","rowLabel":"Pain?"},{"name":"q2_087","label":"Yes","rowLabel":"General Health?"},{"name":"q2_088","label":"Yes","rowLabel":"Doctor or parent instructions?"},{"name":"q2_089","label":"Yes","rowLabel":"Fear the other kids won't like you?"},{"name":"q2_090","label":"Yes","rowLabel":"Friends not around?"}]},{"t":"text","text":"Additional Comment for Selection(s) Above:"},{"t":"textarea","name":"q2_085comment"}] } },
    { n: 21, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (self reported)"},{"t":"text","text":"How often in the last week did you participate in gym/recess ?"},{"t":"radios","name":"q2_091","options":[{"value":1,"label":"Often"},{"value":2,"label":"Sometimes"},{"value":3,"label":"Never or rarely"},{"value":4,"label":"No gym or recess"}]}] }, required: ["q2_091"] },
    { n: 22, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (self reported)"},{"t":"text","text":"How often in the last week did you participate in gym/recess ? (cont.)"},{"t":"text","text":"Was your activity limited by:"},{"t":"checks","items":[{"name":"q2_092","label":"Yes","rowLabel":"Pain?"},{"name":"q2_093","label":"Yes","rowLabel":"General Health?"},{"name":"q2_094","label":"Yes","rowLabel":"Doctor or parent instructions?"},{"name":"q2_095","label":"Yes","rowLabel":"Fear the other kids won't like you?"},{"name":"q2_096","label":"Yes","rowLabel":"Dislike of gym/recess?"},{"name":"q2_097","label":"Yes","rowLabel":"School not in session?"},{"name":"q2_098","label":"Yes","rowLabel":"I don't attend school?"}]},{"t":"text","text":"Additional Comment for Selection(s) Above:"},{"t":"textarea","name":"q2_091comment"}] } },
    { n: 23, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (self reported)"},{"t":"text","text":"Is it easy or hard for you to make friends with kids your own age?"},{"t":"radios","name":"q2_099","options":[{"value":1,"label":"Usually easy"},{"value":2,"label":"Sometimes easy"},{"value":3,"label":"Sometimes hard"},{"value":4,"label":"Usually hard"}]}] }, required: ["q2_099"] },
    { n: 24, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (self reported)"},{"t":"text","text":"How much pain have you had during the last week ?"},{"t":"radios","name":"q2_100","options":[{"value":1,"label":"None"},{"value":2,"label":"Very mild"},{"value":3,"label":"Mild"},{"value":4,"label":"Moderate"},{"value":5,"label":"Severe"},{"value":6,"label":"Very severe"}]}] }, required: ["q2_100"] },
    { n: 25, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (self reported)"},{"t":"text","text":"During the last week , how much did pain interfere with your normal activities (including at home, outside of the home, and at school)?"},{"t":"radios","name":"q2_101","options":[{"value":1,"label":"Not at all"},{"value":2,"label":"A little bit"},{"value":3,"label":"Moderately"},{"value":4,"label":"Quite a bit"},{"value":5,"label":"Extremely"}]}] }, required: ["q2_101"] },
    { n: 26, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (self reported)"},{"t":"text","text":"What expectations do you have for your treatment?"},{"t":"text","text":"As a result of my treatment, I expect:"},{"t":"scale","columns":["Definitely Yes","Probably Yes","Not Sure","Probably Not","Definitely Not"],"rows":[{"name":"q2_102","text":"To have pain relief"},{"name":"q2_103","text":"To look better"},{"name":"q2_104","text":"To feel better about myself"},{"name":"q2_105","text":"To sleep more comfortably"},{"name":"q2_106","text":"To be able to do activities at home"},{"name":"q2_107","text":"To be able to do more at school"},{"name":"q2_108","text":"To be able to do more play or recreational activities (biking, walking, doing things with friends)"},{"name":"q2_109","text":"To be able to do more sports"},{"name":"q2_110","text":"To be free from pain or disability as an adult"}]}] }, required: ["q2_102","q2_103","q2_104","q2_105","q2_106","q2_107","q2_108","q2_109","q2_110"] },
    { n: 27, blocks: { en: [{"t":"text","text":"Adolescent Health Assessment (self reported)"},{"t":"text","text":"If you had to spend the rest of your life with your bone and muscle condition as it is right now , how would you feel about it?"},{"t":"radios","name":"q2_111","options":[{"value":1,"label":"Very satisfied"},{"value":2,"label":"Somewhat satisfied"},{"value":3,"label":"Neutral"},{"value":4,"label":"Somewhat dissatisfied"},{"value":5,"label":"Very dissatisfied"}]}] }, required: ["q2_111"] },
  ],
};
















<div class="podci-form">
    @if (saved) {
    <div class="submitted-banner">
        PODCI questionnaire saved.
    </div>
    }

    @for (page of pages; track page.n) {
    <section class="podci-page">
        @for (block of page.blocks; track $index) {

        @if (block.t === 'text') {
        <p class="stem">{{ text(block.text) }}</p>

        } @else if (block.t === 'radios') {
        <div class="option-list">
            @for (option of block.options; track option.value) {
            <label class="radio-option">
                <input type="radio" [name]="block.name" [value]="option.value"
                    [checked]="selected(block.name, option.value)" (change)="setRadio(block.name, option.value)" />
                {{ text(option.label) }}
            </label>
            }
        </div>

        } @else if (block.t === 'scale') {
        <div class="table-wrap">
            <table class="scale-table">
                <thead>
                    <tr>
                        <th></th>
                        @for (column of block.columns; track $index) {
                        <th>{{ text(column) }}</th>
                        }
                    </tr>
                </thead>
                <tbody>
                    @for (row of block.rows; track row.name) {
                    <tr>
                        <td class="row-label">{{ text(row.text) }}</td>
                        @for (column of block.columns; track $index; let i = $index) {
                        <td class="radio-cell">
                            <input type="radio" [name]="row.name" [value]="i + 1" [attr.aria-label]="column"
                                [checked]="selected(row.name, i + 1)" (change)="setRadio(row.name, i + 1)" />
                        </td>
                        }
                    </tr>
                    }
                </tbody>
            </table>
        </div>

        } @else if (block.t === 'grid') {
        <div class="table-wrap">
            <table class="scale-table">
                <thead>
                    <tr>
                        <th></th>
                        @for (column of block.columns; track $index) {
                        <th>{{ text(column) }}</th>
                        }
                    </tr>
                </thead>
                <tbody>
                    @for (row of block.rows; track row.label) {
                    <tr>
                        <td class="row-label">{{ text(row.label) }}</td>
                        @for (cell of row.cells; track cell.name) {
                        <td class="radio-cell">
                            <div class="yes-no">
                                @for (option of cell.options; track option.value) {
                                <label class="yes-no-option" [class.disabled]="gridDisabled(cell.name)">
                                    <input type="radio" [name]="cell.name" [value]="option.value"
                                        [checked]="selected(cell.name, option.value)"
                                        [disabled]="gridDisabled(cell.name)"
                                        (change)="setGrid(cell.name, option.value)" />
                                    {{ option.label }}
                                </label>
                                }
                            </div>
                        </td>
                        }
                    </tr>
                    }
                </tbody>
            </table>
        </div>

        } @else if (block.t === 'checks') {
        @if (block.items[0]?.rowLabel) {
        <div class="limiter-list">
            @for (item of block.items; track item.name) {
            <label class="limiter-row">
                <span class="limiter-text">{{ text(item.rowLabel) }}</span>
                <span class="limiter-check">
                    <input type="checkbox" [checked]="isChecked(item.name)"
                        (change)="setChecked(item.name, $any($event.target).checked)" />
                    {{ text(item.label) }}
                </span>
            </label>
            }
        </div>
        } @else {
        <div class="region-grid">
            @for (item of block.items; track item.name) {
            <label class="region-option">
                <input type="checkbox" [checked]="isChecked(item.name)"
                    (change)="setChecked(item.name, $any($event.target).checked)" />
                {{ text(item.label) }}
            </label>
            }
        </div>
        }

        } @else if (block.t === 'textarea') {
        <textarea class="comment-box" rows="3" [ngModel]="comment(block.name)"
            (ngModelChange)="setComment(block.name, $event)"></textarea>
        }

        }
    </section>
    }

    @if (saveError) {
    <div class="questionnaire-error">{{ saveError }}</div>
    }

    @if (!hideActions) {
    <div class="podci-actions">
        <button type="button" class="submit-btn" [disabled]="saving" (click)="save()">
            {{ saving ? 'Saving...' : (saved ? 'Save Again' : 'Save') }}
        </button>
    </div>
    }
</div>











:host {
    display: block;
}

.podci-form {
    display: flex;
    flex-direction: column;
    gap: 16px;
}


.submitted-banner {
    padding: 12px 16px;
    background: #eef6f4;
    border: 1px solid #a6d8cf;
    border-radius: 8px;
    color: #1f6b5e;
    font-size: 13px;
    font-weight: 500;
}

.podci-page {
    display: flex;
    flex-direction: column;
    gap: 10px;
    padding: 18px 20px;
    background: #ffffff;
    border: 1px solid #e2e8f0;
    border-radius: 10px;
}

.stem {
    margin: 0;
    color: #1e293b;
    font-size: 14px;
    line-height: 1.5;
}

.option-list {
    display: flex;
    flex-direction: column;
    gap: 6px;
}

.radio-option {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 9px 12px;
    border-radius: 8px;
    font-size: 13px;
    color: #374151;
    cursor: pointer;
    transition: background-color 0.12s ease;
}

.radio-option:hover {
    background: #f8fafc;
}

.radio-option:has(input:checked) {
    background: #eef6f4;
    color: #1f6b5e;
    font-weight: 500;
}

.radio-option input,
.limiter-check input,
.region-option input,
.yes-no-option input {
    width: 16px;
    height: 16px;
    accent-color: #269c96;
    cursor: pointer;
    flex-shrink: 0;
}

.table-wrap {
    overflow-x: auto;
    border: 1px solid #e2e8f0;
    border-radius: 10px;
}

.scale-table {
    width: 100%;
    border-collapse: collapse;
    font-size: 13px;
}

.scale-table th,
.scale-table td {
    padding: 9px 12px;
    border-bottom: 1px solid #eef1f4;
    text-align: center;
}

.scale-table tbody tr:last-child td {
    border-bottom: none;
}

.scale-table th {
    background: #f8fafc;
    color: #475569;
    font-size: 12px;
    font-weight: 700;
}

.scale-table .row-label {
    text-align: left;
    color: #1e293b;
    min-width: 220px;
}

.scale-table tbody tr:hover {
    background: #f8fafc;
}

.scale-table input[type="radio"] {
    width: 16px;
    height: 16px;
    accent-color: #269c96;
    cursor: pointer;
}

.yes-no {
    display: flex;
    justify-content: center;
    gap: 14px;
}

.yes-no-option {
    display: flex;
    align-items: center;
    gap: 5px;
    cursor: pointer;
}

.yes-no-option.disabled {
    color: #9ca3af;
    cursor: not-allowed;
}

.region-grid {
    display: grid;
    grid-template-columns: repeat(3, minmax(0, 1fr));
    gap: 6px 12px;
}

.region-option {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 8px 10px;
    border-radius: 8px;
    font-size: 13px;
    color: #374151;
    cursor: pointer;
}

.region-option:hover {
    background: #f8fafc;
}

.limiter-list {
    display: flex;
    flex-direction: column;
    gap: 2px;
}

.limiter-row {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 16px;
    padding: 8px 12px;
    border-radius: 8px;
    font-size: 13px;
    color: #374151;
    cursor: pointer;
}

.limiter-row:hover {
    background: #f8fafc;
}

.limiter-check {
    display: flex;
    align-items: center;
    gap: 6px;
}

.comment-box {
    width: 100%;
    padding: 10px 12px;
    box-sizing: border-box;
    border: 1px solid #d7dde2;
    border-radius: 8px;
    font-family: inherit;
    font-size: 13px;
}

.comment-box:focus {
    outline: none;
    border-color: #269c96;
    box-shadow: 0 0 0 3px rgba(38, 156, 150, 0.12);
}

.questionnaire-error {
    padding: 12px 14px;
    border: 1px solid #efc5c5;
    border-radius: 8px;
    background: #fff4f4;
    color: #b42318;
    font-size: 13px;
}

.podci-actions {
    display: flex;
    justify-content: flex-end;
}

.submit-btn {
    height: 40px;
    padding: 0 22px;
    color: #ffffff;
    background: #269c96;
    border: 1px solid #1f847f;
    border-radius: 8px;
    font-family: inherit;
    font-size: 14px;
    font-weight: 600;
    cursor: pointer;
    box-shadow: 0 1px 3px rgba(31, 132, 127, 0.25);
    transition: background-color 0.15s ease, box-shadow 0.15s ease;
}

.submit-btn:hover:not(:disabled) {
    background: #1f847f;
    box-shadow: 0 2px 6px rgba(31, 132, 127, 0.3);
}

.submit-btn:disabled {
    background: #b7d4d2;
    border-color: #b7d4d2;
    box-shadow: none;
    cursor: not-allowed;
}

@media (max-width: 700px) {
    .region-grid {
        grid-template-columns: 1fr;
    }
}














import { CommonModule } from '@angular/common';
import { ChangeDetectorRef, Component, OnInit, QueryList, ViewChildren } from '@angular/core';
import { FormsModule } from '@angular/forms';
import { HttpClient } from '@angular/common/http';
import { ActivatedRoute } from '@angular/router';

import { Header } from '../../header/header';
import { SportsQuestionnaire } from '../sports-questionnaire/sports-questionnaire';
import { HipQuestionnaire } from '../hip-questionnaire/hip-questionnaire';
import { PodciQuestionnaire, PodciVariant } from '../podci-questionnaire/podci-questionnaire';
import { UeHistoryQuestionnaire } from '../ue-history-questionnaire/ue-history-questionnaire';
import { HistoryQuestionnaire } from '../history-questionnaire/history-questionnaire';
import { fillName, translate } from '../legacy-i18n';
import { BaselineWho, HistoryVisitQuestionnaire } from '../history-visit-questionnaire/history-visit-questionnaire';

// This is the popup window's content — legacy opens a blank, fullscreen,
// named popup (pickpt.jsp's openQuest()) and targets the picker form's
// submit into it, landing find_quest.jsp (which does the actual visit
// resolution + questionnaire assembly) inside that popup, not the
// original window. We mirror that split exactly: Questionnaire (the
// picker) just opens this route in a popup with the picked visit +
// selections as query params; THIS component does the assemble call and
// renders/saves the result, same as find_quest.jsp would.
//
// Legacy does NOT log the clinician out for this (confirmed by tracing
// the source — no session.invalidate() anywhere in the flow, the Struts
// bridge is @Public and just reuses whatever auth is already on the
// session). So this window is same-origin and shares the browser's
// existing session/cookies with the original window, same as legacy —
// no separate auth handling needed here.

// The subset of assembled codes that render as real, savable forms today.
// Everything else shows a not-implemented placeholder and is excluded
// from Save All's pending count.
const SAVABLE_CODES = new Set([
    'SPORTS', 'HIP', 'PODCI_CH', 'PODCI_AP', 'PODCI_AS', 'NEW_HISTORY', 'UE_HISTORY'
]);

// HISTORY, HISTORY_GAIT, and HISTORY_CONCERNS collapse into ONE rendered
// section (app-history-visit-questionnaire) instead of one-per-code — see
// history-visit-questionnaire.ts for why: they share the same visit-scoped
// items/details/medications backend resource, so splitting them across
// separate components risked two of them racing to PUT-replace the same
// scope and silently dropping each other's edits. Excluded from the
// generic per-code loop; rendered once, up front, when any is present.
const COLLAPSED_HISTORY_CODES = new Set(['HISTORY', 'HISTORY_GAIT', 'HISTORY_CONCERNS']);

interface AssembledQuestionnaire {
    code: string;
    visitId: number;
}

interface SavableQuestionnaireComponent {
    save(): void;
    validate?(): string | null;
    saved: boolean;
}

// Display names for assembled questionnaire codes. The backend only
// returns codes (matching the legacy Static enum names). These are
// QuestionnaireServiceImpl.getDisplayName() verbatim; HISTORY_CONCERNS has
// none in legacy (null — never listed).
const QUESTIONNAIRE_NAMES: Record<string, string> = {
    NEW_HISTORY: 'First Visit History',
    HISTORY: 'General History',
    HISTORY_GAIT: 'Gait History',
    HISTORY_CONCERNS: '',
    UE_HISTORY: 'UE History',
    PODCI_CH: 'PODCI (Child)',
    PODCI_AP: 'PODCI (Adolescent Parent-reported)',
    PODCI_AS: 'PODCI (Adolescent Self-reported)',
    SPORTS: 'Sports',
    HIP: 'Hip'
};

@Component({
    selector: 'app-questionnaire-fill',
    standalone: true,
    imports: [
        CommonModule, FormsModule, Header,
        SportsQuestionnaire, HipQuestionnaire, PodciQuestionnaire,
        HistoryQuestionnaire, HistoryVisitQuestionnaire, UeHistoryQuestionnaire
    ],
    templateUrl: './questionnaire-fill.html',
    styleUrl: './questionnaire-fill.css'
})
export class QuestionnaireFill implements OnInit {
    patientId: number | null = null;
    visitId: number | null = null;
    patientDisplayName = '';
    firstName = '';
    languageCode: 'en' | 'sp' = 'en';

    private selections: string[] = [];
    private language = 'en';

    assembling = false;
    assembleError = '';
    assembledCodes: string[] | null = null;
    // Per-code visit the backend resolved (find_quest.jsp: last-match coincident visit, Gait-visit
    // retarget, Sports -> attended RUNNING visit). Falls back to the picked visit.
    private codeVisitIds = new Map<string, number>();
    private assembledList: AssembledQuestionnaire[] = [];

    @ViewChildren('sectionCmp') private sectionCmps!: QueryList<SavableQuestionnaireComponent>;

    savingAll = false;
    saveAllError = '';
    allSaved = false;
    private pendingSaveCount = 0;
    private failedSaveNames: string[] = [];

    constructor(
        private http: HttpClient,
        private cdr: ChangeDetectorRef,
        private route: ActivatedRoute
    ) {}

    ngOnInit(): void {
        const params = this.route.snapshot.queryParamMap;
        this.patientId = params.get('patientId') ? Number(params.get('patientId')) : null;
        this.visitId = params.get('visitId') ? Number(params.get('visitId')) : null;
        this.patientDisplayName = params.get('patientName') ?? '';
        this.firstName = params.get('firstName') ?? '';
        this.selections = (params.get('selections') ?? '').split(',').filter(Boolean);
        this.language = params.get('language') ?? 'en';
        this.languageCode = this.language === 'sp' ? 'sp' : 'en';

        this.assemble();
    }

    private assemble(): void {
        if (!this.patientId || !this.visitId || this.selections.length === 0) {
            this.assembleError = 'Missing patient, visit, or questionnaire selection.';
            return;
        }

        this.assembling = true;
        this.assembleError = '';
        this.assembledCodes = null;
        this.savingAll = false;
        this.saveAllError = '';
        this.allSaved = false;

        this.http.post<AssembledQuestionnaire[]>(
            `/api/patients/${this.patientId}/questionnaire-picker/assemble`,
            {
                visitId: this.visitId,
                selections: this.selections,
                language: this.language
            }
        ).subscribe({
            next: assembled => {
                this.assembling = false;
                this.assembledList = assembled;
                this.codeVisitIds = new Map(assembled.map(item => [item.code, item.visitId]));
                this.assembledCodes = assembled.map(item => item.code);
                this.cdr.markForCheck();
            },
            error: error => {
                console.error('Unable to assemble questionnaire list:', error);
                this.assembling = false;
                this.assembleError = 'Unable to assemble the questionnaire list. Please try again.';
                this.cdr.markForCheck();
            }
        });
    }

    retry(): void {
        this.assemble();
    }

    // Legacy wording for the fill-page level texts, in the session language where legacy has Spanish.
    text(key: string): string {
        return fillName(translate(key, this.languageCode), this.firstName, this.languageCode === 'sp' ? 'su niño(a)' : 'your child');
    }

    visitIdFor(code: string): number | null {
        return this.codeVisitIds.get(code) ?? this.visitId;
    }

    // The visit-history form saves items/details/medications for ONE visit. All history codes share
    // the same visit in practice (the whole bundle is retargeted together); use the anchor's.
    get historyVisitId(): number | null {
        const anchor = this.historyAnchor;
        return anchor ? this.visitIdFor(anchor) : this.visitId;
    }

    questionnaireName(code: string): string {
        return QUESTIONNAIRE_NAMES[code] ?? code;
    }

    podciVariant(code: string | null): PodciVariant {
        if (code === 'PODCI_AP') return 'AP';
        if (code === 'PODCI_AS') return 'AS';
        return 'CH';
    }

    // Which visit-history page groups the assembled list asks for (QuestionnaireServiceImpl.getPosition):
    // HISTORY = who + general; NEW_HISTORY = general only (its "who" pages are the baseline pages);
    // HISTORY_GAIT / HISTORY_CONCERNS add their own pages.
    private has(code: string): boolean {
        return !!this.assembledCodes && this.assembledCodes.includes(code);
    }

    get showWho(): boolean { return this.has('HISTORY'); }
    get showGeneral(): boolean { return this.has('HISTORY') || this.has('NEW_HISTORY'); }
    get showGait(): boolean { return this.has('HISTORY_GAIT'); }
    get showConcerns(): boolean { return this.has('HISTORY_CONCERNS'); }

    // The visit-history form is rendered once, at the position of the first history code — or right
    // after the first-visit baseline pages when NEW_HISTORY is present, as in legacy.
    get historyAnchor(): string | null {
        if (this.has('NEW_HISTORY')) return 'NEW_HISTORY';
        return (this.assembledCodes ?? []).find(code => COLLAPSED_HISTORY_CODES.has(code)) ?? null;
    }

    get visibleAssembledCodes(): string[] {
        const anchor = this.historyAnchor;
        return (this.assembledCodes ?? []).filter(
            code => !COLLAPSED_HISTORY_CODES.has(code) || code === anchor);
    }

    get hasHistoryVisitSection(): boolean {
        return this.historyAnchor !== null;
    }

    // begin-self.jsp: the patient-completed PODCI must be handed over to the patient first.
    get hasSelfPodci(): boolean {
        return this.has('PODCI_AS');
    }

    baseline: BaselineWho | null = null;

    onBaselineChange(who: BaselineWho): void {
        this.baseline = who;
    }

    // show-questionnaires.jsp lists every assembled questionnaire that has a display name, in order.
    get overviewNames(): string[] {
        return (this.assembledCodes ?? []).map(code => this.questionnaireName(code)).filter(name => !!name);
    }

    get hasSavableSections(): boolean {
        return this.hasHistoryVisitSection
            || (!!this.assembledCodes && this.assembledCodes.some(code => SAVABLE_CODES.has(code)));
    }

    saveAll(): void {
        if (!this.assembledCodes || this.savingAll) {
            return;
        }

        // Every section is (re)sent on each Save: their saves replace what was stored, so this is safe to
        // repeat, and it means a change made after an earlier Save (or a retry after a failure) is never skipped.
        const components = this.sectionCmps.toArray();
        if (components.length === 0) {
            return;
        }

        // Legacy's per-page check_ok() blocks the page from being submitted when required answers are
        // missing; with everything on one page, block the whole save the same way.
        const invalid = components.map(component => component.validate?.() ?? null).find(message => !!message);
        if (invalid) {
            this.saveAllError = invalid;
            this.allSaved = false;
            this.cdr.markForCheck();
            return;
        }

        this.savingAll = true;
        this.saveAllError = '';
        this.allSaved = false;
        this.pendingSaveCount = components.length;
        this.failedSaveNames = [];

        components.forEach(component => component.save());
    }

    onSectionSaved(): void {
        this.pendingSaveCount--;
        this.finishSaveAllIfDone();
    }

    onSectionSaveFailed(code: string): void {
        this.failedSaveNames.push(this.questionnaireName(code));
        this.pendingSaveCount--;
        this.finishSaveAllIfDone();
    }

    private finishSaveAllIfDone(): void {
        if (this.pendingSaveCount > 0) {
            return;
        }
        this.savingAll = false;
        if (this.failedSaveNames.length > 0) {
            this.saveAllError = `Unable to save: ${this.failedSaveNames.join(', ')}. Please retry.`;
            this.allSaved = false;
        } else {
            this.saveAllError = '';
            this.allSaved = true;
            this.recordTracking();
        }
        this.cdr.markForCheck();
    }

    // Legacy writes a q_track row for every questionnaire page the visit reached (wasAsked() in the reports
    // reads them). The backend expands the assembled list into those rows and skips ones already there.
    private recordTracking(): void {
        if (!this.patientId || this.assembledList.length === 0) return;
        this.http.post(`/api/patients/${this.patientId}/questionnaire-tracking`, this.assembledList).subscribe({
            error: error => console.error('Unable to record questionnaire tracking:', error)
        });
    }

    closeWindow(): void {
        window.close();
    }
}















<div class="fill-page">
    <app-header></app-header>

    <main class="fill-content">
        @if (assembling) {
        <div class="fill-card fill-status">
            <h2>Assembling Questionnaires…</h2>
            <p class="loading-text">
                Working out which questionnaires apply
                @if (patientDisplayName) {
                for {{ patientDisplayName }}
                }…
            </p>
        </div>
        } @else if (assembleError) {
        <div class="fill-card fill-status">
            <h2>Something went wrong</h2>
            <div class="questionnaire-error">{{ assembleError }}</div>
            <div class="fill-actions">
                <button type="button" class="start-btn" (click)="retry()">Retry</button>
            </div>
        </div>
        } @else if (assembledCodes && assembledCodes.length === 0) {
        <div class="fill-card fill-status">
            <h2>Questionnaires to Complete</h2>
            <div class="empty-note">
                No questionnaires were assembled from this selection.
            </div>
        </div>
        } @else if (assembledCodes) {
        <div class="fill-heading">
            <h1>STOP</h1>
            <p>The following questionnaires will be administered in the given order:</p>
            <ul class="overview-list">
                @for (name of overviewNames; track $index) {
                <li>{{ name }}</li>
                }
            </ul>
        </div>

        @if (saveAllError) {
        <div class="questionnaire-error sticky-banner">{{ saveAllError }}</div>
        }
        @if (allSaved) {
        <div class="submitted-banner sticky-banner">
            <strong>{{ text('Thank You') }}</strong><br>
            {{ text('Thank You for Filling Out the Questionnaire. This information helps us get the big picture and provide the best possible care.') }}<br>
            {{ text('Please tell a member of our staff that you have finished.') }}
        </div>
        }

        <div class="fill-sections">
            @for (code of visibleAssembledCodes; track code) {
            @if (code === 'PODCI_AS') {
            <section class="assembled-section stop-section">
                <h3 class="assembled-section-title">STOP</h3>
                <p class="stop-text">{{ text('This section should be filled out by the patient.') }}</p>
            </section>
            }
            <section class="assembled-section">
                @if (questionnaireName(code)) {
                <h3 class="assembled-section-title">{{ questionnaireName(code) }}</h3>
                }

                @if (code === historyAnchor) {
                @if (code === 'NEW_HISTORY') {
                <app-history-questionnaire #sectionCmp [patientId]="patientId" [firstName]="firstName" [language]="languageCode"
                    [hideActions]="true" (baselineChange)="onBaselineChange($event)"
                    (saveComplete)="onSectionSaved()"
                    (saveFailed)="onSectionSaveFailed('NEW_HISTORY')"></app-history-questionnaire>
                }
                <app-history-visit-questionnaire #sectionCmp [patientId]="patientId" [visitId]="historyVisitId"
                    [firstName]="firstName" [language]="languageCode" [showWho]="showWho" [showGeneral]="showGeneral"
                    [showGait]="showGait" [showConcerns]="showConcerns" [copyWho]="showWho ? null : baseline"
                    [hideActions]="true" (saveComplete)="onSectionSaved()"
                    (saveFailed)="onSectionSaveFailed('HISTORY')"></app-history-visit-questionnaire>
                @if (code === 'NEW_HISTORY') {
                <div class="return-intro">
                    <h4>End of Part One</h4>
                    <p>This is the end of the history questionnaire. On future visits, you will only have to give us updates.</p>
                    <p>You will now be taken into the regular questionnaire that we use for each visit. Your answers will help us better treat the gait problems that brought you to our lab.</p>
                </div>
                }
                } @else if (code === 'SPORTS') {
                <app-sports-questionnaire #sectionCmp [patientId]="patientId" [visitId]="visitIdFor(code)"
                    [hideActions]="true" (saveComplete)="onSectionSaved()"
                    (saveFailed)="onSectionSaveFailed(code)"></app-sports-questionnaire>
                } @else if (code === 'UE_HISTORY') {
                <app-ue-history-questionnaire #sectionCmp [patientId]="patientId" [visitId]="visitIdFor(code)"
                    [hideActions]="true" (saveComplete)="onSectionSaved()"
                    (saveFailed)="onSectionSaveFailed(code)"></app-ue-history-questionnaire>
                } @else if (code === 'HIP') {
                <app-hip-questionnaire #sectionCmp [patientId]="patientId" [visitId]="visitIdFor(code)" [hideActions]="true"
                    (saveComplete)="onSectionSaved()"
                    (saveFailed)="onSectionSaveFailed(code)"></app-hip-questionnaire>
                } @else if (code === 'PODCI_CH' || code === 'PODCI_AP' || code === 'PODCI_AS') {
                <app-podci-questionnaire #sectionCmp [patientId]="patientId" [visitId]="visitIdFor(code)"
                    [firstName]="firstName" [language]="languageCode" [variant]="podciVariant(code)"
                    [hideActions]="true"
                    (saveComplete)="onSectionSaved()"
                    (saveFailed)="onSectionSaveFailed(code)"></app-podci-questionnaire>
                } @else {
                <div class="not-implemented-step">
                    This questionnaire ({{ questionnaireName(code) }}) hasn't been built yet.
                </div>
                }
            </section>
            }
        </div>

        <div class="fill-footer">
            <button type="button" class="secondary-btn" (click)="closeWindow()">Close Window</button>
            <button type="button" class="start-btn" [disabled]="!hasSavableSections || savingAll" (click)="saveAll()">
                {{ savingAll ? 'Saving...' : 'Save' }}
            </button>
        </div>
        }
    </main>
</div>














:host {
    display: block;
    min-height: 100vh;
    background: #f7f9fa;
    color: #263238;
}

.fill-page {
    min-height: 100vh;
    background: #f7f9fa;
}

.fill-content {
    max-width: 900px;
    margin: 0 auto;
    padding: 28px 30px 60px;
    box-sizing: border-box;
    display: flex;
    flex-direction: column;
    gap: 18px;
}

.fill-heading {
    margin-bottom: 4px;
}

.fill-heading h1 {
    margin: 0;
    font-size: 26px;
    font-weight: 700;
    color: #1f2937;
}

.fill-heading p {
    margin: 6px 0 0;
    font-size: 14px;
    color: #6b7280;
}

.fill-card {
    background: #ffffff;
    border: 1px solid #e4e8eb;
    border-radius: 10px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.04);
    padding: 32px;
    box-sizing: border-box;
}

.fill-status {
    max-width: 560px;
    margin: 60px auto 0;
    text-align: center;
}

.fill-status h2 {
    margin: 0 0 10px;
    font-size: 20px;
    font-weight: 700;
    color: #1f2937;
}

.loading-text {
    margin: 0;
    color: #6b7280;
    font-size: 14px;
}

.empty-note {
    padding: 14px 16px;
    background: #f7f9fa;
    border: 1px dashed #d8e1e4;
    border-radius: 8px;
    color: #6b7280;
    font-size: 14px;
}

.fill-actions {
    margin-top: 20px;
    display: flex;
    justify-content: center;
}

.sticky-banner {
    position: sticky;
    top: 0;
    z-index: 5;
}

.submitted-banner {
    padding: 12px 16px;
    background: #eef6f4;
    border: 1px solid #a6d8cf;
    border-radius: 8px;
    color: #1f6b5e;
    font-size: 13px;
    font-weight: 500;
}

.questionnaire-error {
    padding: 12px 14px;
    border: 1px solid #efc5c5;
    border-radius: 8px;
    background: #fff4f4;
    color: #b42318;
    font-size: 13px;
}

.fill-sections {
    display: flex;
    flex-direction: column;
    gap: 18px;
}

.assembled-section {
    padding: 20px 22px;
    background: #fafbfc;
    border: 1px solid #eef1f4;
    border-radius: 12px;
}

.assembled-section-title {
    margin: 0 0 14px;
    padding-left: 12px;
    border-left: 4px solid #269c96;
    color: #1e293b;
    font-size: 16px;
    font-weight: 700;
}

.not-implemented-step {
    padding: 18px;
    background: #f7f9fa;
    border: 1px dashed #d8e1e4;
    border-radius: 6px;
    color: #6b7280;
    font-size: 14px;
    line-height: 1.5;
}

.fill-footer {
    position: sticky;
    bottom: 0;
    display: flex;
    justify-content: flex-end;
    gap: 10px;
    padding: 16px 0;
    margin-top: 4px;
    background: linear-gradient(to top, #f7f9fa 60%, rgba(247, 249, 250, 0));
}

.start-btn {
    min-width: 140px;
    height: 46px;
    padding: 0 24px;
    border: none;
    border-radius: 8px;
    background: #269c96;
    color: #ffffff;
    font-family: inherit;
    font-size: 15px;
    font-weight: 600;
    cursor: pointer;
    box-shadow: 0 1px 3px rgba(31, 132, 127, 0.25);
    transition: background-color 0.15s ease, box-shadow 0.15s ease;
}

.start-btn:hover:not(:disabled) {
    background: #218a85;
    box-shadow: 0 2px 6px rgba(31, 132, 127, 0.3);
}

.start-btn:disabled {
    background: #b7d4d2;
    box-shadow: none;
    cursor: not-allowed;
}

.secondary-btn {
    min-width: 100px;
    height: 46px;
    padding: 0 20px;
    border: 1px solid #d7dde2;
    border-radius: 8px;
    background: #ffffff;
    color: #30363b;
    font-family: inherit;
    font-size: 14px;
    font-weight: 600;
    cursor: pointer;
}

.secondary-btn:hover {
    background: #f7f9fa;
}

@media (max-width: 700px) {
    .fill-content {
        padding: 20px 16px 48px;
    }
}

.stop-section {
    text-align: center;
    background: #fff9f0;
    border-color: #f0dcc0;
}

.stop-text {
    margin: 0;
    color: #1e3a5f;
    font-size: 14px;
    font-weight: 600;
}

.return-intro {
    padding: 14px 16px;
    margin: 4px 0;
    background: #f8fafc;
    border: 1px solid #e2e8f0;
    border-radius: 8px;
    color: #374151;
    font-size: 13px;
    line-height: 1.5;
}

.return-intro h4 {
    margin: 0 0 6px;
    color: #1e293b;
    font-size: 14px;
}

.return-intro p {
    margin: 0 0 6px;
}

.overview-list {
    margin: 8px 0 0;
    padding-left: 22px;
    font-size: 14px;
    color: #374151;
}












import { CommonModule } from '@angular/common';
import { ChangeDetectorRef, Component, EventEmitter, Input, OnChanges, Output, SimpleChanges } from '@angular/core';
import { FormsModule } from '@angular/forms';
import { HttpClient } from '@angular/common/http';

// SPORTS1-9, migrated from the legacy Sports questionnaire
// (nmquestionnaire.question / question_text / answer / answer_text,
// questionnaire_id=1). Confirmed by the legacy session to be a straight
// linear chain with no branching, so it's modeled as a flat form here
// rather than a generic question-graph engine.
//
// Persists into the restored nmquestionnaire schema (session +
// patient_answer), matching legacy's actual storage exactly, not a
// simplified stand-in — approved by the user 2026-09-21 after restoring
// the schema from the legacy backup. Choice answers are sent as their
// label text; the backend resolves the matching answer_id via
// answer_text (seeded to match these labels exactly).
//
// Takes visitId (resolved once at picker time), not a date — legacy's
// Sports save path (BridgeService.gaitToQuestionnaire) resolves the same
// currVisitID session value everything else does, by primary key, never
// a fresh (patient_id, date) lookup. See history-visit-questionnaire.ts
// for the full citation.
export interface SportsAnswers {
    reportingPerson: string;
    physicalTherapy: string;
    competitiveSports: string;
    yearsInSport: string;
    daysPerWeek: string;
    orthotics: string;
    painLocations: string[];
    mileTimeKnown: 'unknown' | 'known' | '';
    mileTime: string;
    goals: string;
}

@Component({
    selector: 'app-sports-questionnaire',
    standalone: true,
    imports: [CommonModule, FormsModule],
    templateUrl: './sports-questionnaire.html',
    styleUrl: './sports-questionnaire.css'
})
export class SportsQuestionnaire implements OnChanges {
    @Input() patientId: number | null = null;
    @Input() visitId: number | null = null;
    @Input() hideActions = false;
    @Output() saveComplete = new EventEmitter<void>();
    @Output() saveFailed = new EventEmitter<string>();

    readonly physicalTherapyOptions = [
        '3 or more times a week',
        '2 times a week',
        'Once a week',
        'Less than once a week',
        'Not currently'
    ];

    readonly yearsInSportOptions = [
        'Less than one year',
        '1-2 years',
        '3-4 years',
        '5 years or more'
    ];

    readonly daysPerWeekOptions = ['none', '1', '2', '3', '4', '5', '6', '7'];

    readonly orthoticsOptions = [
        'None',
        'Over the counter orthotics',
        'Custom fitted orthotics'
    ];

    readonly painLocationOptions = [
        'None',
        'Back',
        'R Hip', 'L Hip',
        'R Knee', 'L Knee',
        'R Ankle', 'L Ankle',
        'R Foot', 'L Foot',
        'R Thigh', 'L Thigh',
        'R Lower leg', 'L Lower leg'
    ];

    answers: SportsAnswers = this.createEmptyAnswers();

    saving = false;
    saveError = '';
    saved = false;

    constructor(private http: HttpClient, private cdr: ChangeDetectorRef) {}

    ngOnChanges(changes: SimpleChanges): void {
        if (!(changes['patientId'] || changes['visitId'])) return;
        this.answers = this.createEmptyAnswers();
        this.saved = false;
        this.saveError = '';
        this.preload();
    }

    // Shows the visit's own saved answers (legacy re-shows what was stored when the pages are reopened).
    // No "last visit" fallback: nothing in the legacy source we can read actually applies populate='last'.
    private preload(): void {
        if (!this.patientId || !this.visitId) {
            return;
        }
        const visitId = this.visitId;
        this.http.get<Partial<Record<keyof SportsAnswers, string | string[] | null>>>(
            `/api/patients/${this.patientId}/sports-questionnaire/${visitId}`
        ).subscribe({
            next: saved => {
                if (visitId !== this.visitId || !saved) return;
                const text = (value: string | string[] | null | undefined) => typeof value === 'string' ? value : '';
                const known = saved.mileTimeKnown;
                this.answers = {
                    reportingPerson: text(saved.reportingPerson),
                    physicalTherapy: text(saved.physicalTherapy),
                    competitiveSports: text(saved.competitiveSports),
                    yearsInSport: text(saved.yearsInSport),
                    daysPerWeek: text(saved.daysPerWeek),
                    orthotics: text(saved.orthotics),
                    painLocations: Array.isArray(saved.painLocations) ? [...saved.painLocations] : [],
                    mileTimeKnown: known === 'known' || known === 'unknown' ? known : '',
                    mileTime: text(saved.mileTime),
                    goals: text(saved.goals)
                };
                this.cdr.markForCheck();
            },
            error: () => {}
        });
    }

    private createEmptyAnswers(): SportsAnswers {
        return {
            reportingPerson: '',
            physicalTherapy: '',
            competitiveSports: '',
            yearsInSport: '',
            daysPerWeek: '',
            orthotics: '',
            painLocations: [],
            mileTimeKnown: '',
            mileTime: '',
            goals: ''
        };
    }

    isPainSelected(location: string): boolean {
        return this.answers.painLocations.includes(location);
    }

    // Legacy (SPORTS6GroupContainer + ChoiceDisablesOtherGroupContainer.jsp): both "None" and "Back" carry
    // the "single" tag; checking None disables every choice that is NOT tagged "single" — i.e. all the
    // R/L body-part boxes — but leaves Back enabled. Disabled inputs aren't submitted, so they're cleared.
    isPainDisabled(location: string): boolean {
        return location !== 'None' && location !== 'Back' && this.isPainSelected('None');
    }

    togglePainLocation(location: string): void {
        if (this.isPainDisabled(location)) {
            return;
        }

        const selected = this.isPainSelected(location);
        let next = selected
            ? this.answers.painLocations.filter(item => item !== location)
            : [...this.answers.painLocations, location];

        if (location === 'None' && !selected) {
            next = next.filter(item => item === 'None' || item === 'Back');
        }

        this.answers.painLocations = next;
    }

    // /data/questionnaire's group pages disable "Next" until every required question has an answer
    // (before.jsp / after.jsp / question/snippets/after.jsp; all nine Sports questions are required):
    // a choice for the radio questions (mile time needs only the radio — the "Time" text is never
    // checked), at least one box for pain (None or Back count), and real text — non-empty and not the
    // "Tap to Enter Text" placeholder — for reporting person, competitive sports and goals.
    validate(): string | null {
        const text = (value: string) => value.trim() !== '' && value.trim() !== 'Tap to Enter Text';
        const a = this.answers;
        const complete = text(a.reportingPerson) && a.physicalTherapy !== '' && text(a.competitiveSports)
            && a.yearsInSport !== '' && a.daysPerWeek !== '' && a.orthotics !== ''
            && a.painLocations.length > 0 && a.mileTimeKnown !== '' && text(a.goals);
        return complete ? null : 'You must answer above before submitting.';
    }

    save(): void {
        if (!this.patientId || !this.visitId) {
            this.saveComplete.emit();
            return;
        }

        this.saving = true;
        this.saveError = '';

        this.http.post(
            `/api/patients/${this.patientId}/sports-questionnaire`,
            {
                visitId: this.visitId,
                ...this.answers
            }
        ).subscribe({
            next: () => {
                this.saving = false;
                this.saved = true;
                this.cdr.markForCheck();
                this.saveComplete.emit();
            },
            error: error => {
                console.error('Unable to save Sports questionnaire:', error);
                this.saving = false;
                this.saveError = 'Unable to save. Please try again.';
                this.cdr.markForCheck();
                this.saveFailed.emit(this.saveError);
            }
        });
    }
}












<div class="sports-form">
        @if (saved) {
            <div class="submitted-banner">
                Sports questionnaire saved.
            </div>
        }
        @if (saveError) {
            <div class="questionnaire-error">{{ saveError }}</div>
        }

        <div class="sports-body">
            <div class="question-block">
                <label class="question-label" for="reportingPerson">
                    Please give the name of the person answering these questions.
                </label>
                <input id="reportingPerson" type="text" [(ngModel)]="answers.reportingPerson" />
            </div>

            <div class="question-block">
                <span class="question-label">Do you currently have physical therapy?</span>
                <div class="option-list">
                    @for (option of physicalTherapyOptions; track option) {
                        <label class="radio-option">
                            <input type="radio" name="physicalTherapy" [value]="option"
                                [(ngModel)]="answers.physicalTherapy" />
                            {{ option }}
                        </label>
                    }
                </div>
            </div>

            <div class="question-block">
                <label class="question-label" for="competitiveSports">
                    In the past year which competitive sports have you participated in?
                </label>
                <textarea id="competitiveSports" rows="2" [(ngModel)]="answers.competitiveSports"></textarea>
            </div>

            <div class="question-block">
                <span class="question-label">
                    How many years have you participated in your primary competitive sport?
                </span>
                <div class="option-list">
                    @for (option of yearsInSportOptions; track option) {
                        <label class="radio-option">
                            <input type="radio" name="yearsInSport" [value]="option"
                                [(ngModel)]="answers.yearsInSport" />
                            {{ option }}
                        </label>
                    }
                </div>
            </div>

            <div class="question-block">
                <span class="question-label">
                    How many days in a typical week do you participate in competitive sports?
                </span>
                <div class="option-list option-list-inline">
                    @for (option of daysPerWeekOptions; track option) {
                        <label class="radio-option">
                            <input type="radio" name="daysPerWeek" [value]="option"
                                [(ngModel)]="answers.daysPerWeek" />
                            {{ option }}
                        </label>
                    }
                </div>
            </div>

            <div class="question-block">
                <span class="question-label">Do you currently wear orthotics?</span>
                <div class="option-list">
                    @for (option of orthoticsOptions; track option) {
                        <label class="radio-option">
                            <input type="radio" name="orthotics" [value]="option"
                                [(ngModel)]="answers.orthotics" />
                            {{ option }}
                        </label>
                    }
                </div>
            </div>

            <div class="question-block">
                <span class="question-label">
                    Do you have current pain? (None must be unchecked if you wish to select anything else)
                </span>
                <div class="option-list option-list-grid">
                    @for (option of painLocationOptions; track option) {
                        <label class="radio-option" [class.disabled-choice]="isPainDisabled(option)">
                            <input type="checkbox" [checked]="isPainSelected(option)"
                                [disabled]="isPainDisabled(option)"
                                (change)="togglePainLocation(option)" />
                            {{ option }}
                        </label>
                    }
                </div>
            </div>

            <div class="question-block">
                <span class="question-label">What is your best mile time?</span>
                <div class="option-list">
                    <label class="radio-option">
                        <input type="radio" name="mileTimeKnown" value="unknown"
                            [(ngModel)]="answers.mileTimeKnown" />
                        Unknown
                    </label>
                    <label class="radio-option mile-time-option">
                        <input type="radio" name="mileTimeKnown" value="known"
                            [(ngModel)]="answers.mileTimeKnown" />
                        Time:
                        <input type="text" class="inline-input"
                            [(ngModel)]="answers.mileTime" />
                    </label>
                </div>
            </div>

            <div class="question-block">
                <label class="question-label" for="goals">What are your current goals?</label>
                <textarea id="goals" rows="2" [(ngModel)]="answers.goals"></textarea>
            </div>
        </div>

        @if (!hideActions) {
        <div class="sports-actions">
            <button type="button" class="submit-btn" [disabled]="saving || saved" (click)="save()">
                {{ saving ? 'Saving...' : (saved ? 'Saved' : 'Save') }}
            </button>
        </div>
        }
</div>
















:host {
    display: block;
}

.sports-form {
    display: flex;
    flex-direction: column;
    gap: 16px;
}

.submitted-banner {
    padding: 12px 16px;
    background: #eef6f4;
    border: 1px solid #a6d8cf;
    border-radius: 8px;
    color: #1f6b5e;
    font-size: 13px;
    font-weight: 500;
}

.questionnaire-error {
    padding: 12px 14px;
    border: 1px solid #efc5c5;
    border-radius: 8px;
    background: #fff4f4;
    color: #b42318;
    font-size: 13px;
}

.sports-body {
    display: flex;
    flex-direction: column;
    gap: 14px;
}

.question-block {
    display: flex;
    flex-direction: column;
    gap: 12px;
    padding: 18px 20px;
    background: #ffffff;
    border: 1px solid #e2e8f0;
    border-radius: 10px;
    transition: border-color 0.15s ease, box-shadow 0.15s ease;
}

.question-block:hover {
    border-color: #cbd5e1;
    box-shadow: 0 1px 6px rgba(15, 23, 42, 0.05);
}

.question-block:focus-within {
    border-color: #269c96;
    box-shadow: 0 0 0 3px rgba(38, 156, 150, 0.12);
}

.question-label {
    font-size: 14px;
    font-weight: 600;
    color: #1e293b;
    line-height: 1.4;
}

.question-block input[type="text"],
.question-block textarea {
    width: 100%;
    padding: 10px 14px;
    box-sizing: border-box;
    border: 1px solid #d7dde2;
    border-radius: 8px;
    background: #ffffff;
    color: #263238;
    font-family: inherit;
    font-size: 13px;
}

.question-block input[type="text"]:focus,
.question-block textarea:focus {
    outline: none;
    border-color: #269c96;
    box-shadow: 0 0 0 3px rgba(38, 156, 150, 0.12);
}

.option-list {
    display: flex;
    flex-direction: column;
    gap: 6px;
}

.option-list-inline {
    flex-direction: row;
    flex-wrap: wrap;
    gap: 8px;
}

.option-list-grid {
    display: flex;
    flex-direction: column;
    gap: 6px;
}

.radio-option {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 9px 12px;
    border-radius: 8px;
    font-size: 13px;
    color: #374151;
    cursor: pointer;
    transition: background-color 0.12s ease;
}

.radio-option:hover {
    background: #f8fafc;
}

.radio-option:has(input:checked) {
    background: #eef6f4;
    color: #1f6b5e;
    font-weight: 500;
}

.radio-option input[type="radio"],
.radio-option input[type="checkbox"] {
    width: 16px;
    height: 16px;
    accent-color: #269c96;
    cursor: pointer;
    flex-shrink: 0;
}

.option-list-inline .radio-option {
    padding: 8px 14px;
    border: 1px solid #e2e8f0;
}

.option-list-inline .radio-option:has(input:checked) {
    border-color: #269c96;
}

.mile-time-option {
    gap: 10px;
}

.inline-input {
    width: 100px;
    height: 32px;
    padding: 0 10px;
    box-sizing: border-box;
    border: 1px solid #d7dde2;
    border-radius: 6px;
    font-family: inherit;
    font-size: 13px;
}

.inline-input:disabled {
    background: #f3f4f6;
    color: #9ca3af;
}

.sports-actions {
    display: flex;
    justify-content: flex-end;
    padding-top: 4px;
}

.submit-btn {
    height: 40px;
    padding: 0 22px;
    color: #ffffff;
    background: #269c96;
    border: 1px solid #1f847f;
    border-radius: 8px;
    font-family: inherit;
    font-size: 14px;
    font-weight: 600;
    cursor: pointer;
    box-shadow: 0 1px 3px rgba(31, 132, 127, 0.25);
    transition: background-color 0.15s ease, box-shadow 0.15s ease;
}

.submit-btn:hover:not(:disabled) {
    background: #1f847f;
    box-shadow: 0 2px 6px rgba(31, 132, 127, 0.3);
}

.submit-btn:disabled {
    background: #b7d4d2;
    border-color: #b7d4d2;
    box-shadow: none;
    cursor: not-allowed;
}

.radio-option.disabled-choice {
    color: #9ca3af;
    cursor: not-allowed;
}












import { CommonModule } from '@angular/common';
import { ChangeDetectorRef, Component, EventEmitter, Input, OnChanges, Output } from '@angular/core';
import { HttpClient } from '@angular/common/http';

// UE History (Questionnaire.UE_HISTORY_ID = 7): three Struts pages, in this order — UEH01, UEH02 (three
// questions on one page), UEH05. Text is the English question_text / answer_text of the legacy data
// (nmquestionnaire, questions 54-58); legacy ALWAYS renders it in English (struts_redirect.jsp sets
// language.id = 1 for the /data/questionnaire pages), so there is no Spanish here by design.
// Answers are sent as the canonical answer label; the backend resolves the answer ids.

interface SingleChoice {
    key: 'handlesObjects' | 'dresses' | 'bathes' | 'toilets';
    question: string;
    options: string[];
}

const DEPENDENCE = ['Yes, independently', 'Yes but needs some assistance', 'No, is dependent'];

@Component({
    selector: 'app-ue-history-questionnaire',
    standalone: true,
    imports: [CommonModule],
    templateUrl: './ue-history-questionnaire.html',
    styleUrl: './ue-history-questionnaire.css'
})
export class UeHistoryQuestionnaire implements OnChanges {
    @Input() patientId: number | null = null;
    @Input() visitId: number | null = null;
    @Input() hideActions = false;
    @Output() saveComplete = new EventEmitter<void>();
    @Output() saveFailed = new EventEmitter<string>();

    // UEH01
    readonly handsQuestion: SingleChoice = {
        key: 'handlesObjects',
        question: 'My child handles objects with both hands',
        options: ['Yes', 'Yes but with difficulty', 'No RIGHT hand only', 'No LEFT hand only']
    };

    // UEH02
    readonly selfCareQuestions: SingleChoice[] = [
        { key: 'dresses', question: 'My child is able to dress him/herself', options: DEPENDENCE },
        { key: 'bathes', question: 'My child is able to bathe him/herself', options: DEPENDENCE },
        { key: 'toilets', question: 'My child is able to use the toilet him/herself', options: DEPENDENCE }
    ];

    // UEH05
    readonly concernsQuestion = 'Concerns with arm /hand';
    readonly concernOptions = [
        'Elbow flexed', 'Wrist flexed', 'Thumb in palm', 'Hand fisted',
        'Weak grasp', 'Palm down', 'Neglects arm', 'Hard to get in a sleeve'
    ];

    answers: Record<string, string | null> = {};
    concerns = new Set<string>();

    saving = false;
    saveError = '';
    saved = false;

    constructor(private http: HttpClient, private cdr: ChangeDetectorRef) {}

    ngOnChanges(): void {
        this.answers = {};
        this.concerns = new Set<string>();
        this.saved = false;
        this.saveError = '';
        this.preload();
    }

    private preload(): void {
        if (!this.patientId || !this.visitId) {
            return;
        }
        const visitId = this.visitId;
        this.http.get<Record<string, string | string[] | null>>(
            `/api/patients/${this.patientId}/ue-history-questionnaire/${visitId}`
        ).subscribe({
            next: saved => {
                if (visitId !== this.visitId) return;
                for (const key of ['handlesObjects', 'dresses', 'bathes', 'toilets']) {
                    const value = saved?.[key];
                    if (typeof value === 'string' && value) this.answers[key] = value;
                }
                const chosen = saved?.['armHandConcerns'];
                if (Array.isArray(chosen)) this.concerns = new Set(chosen);
                this.cdr.markForCheck();
            },
            error: () => {}
        });
    }

    choose(key: string, value: string): void {
        this.answers[key] = value;
    }

    toggleConcern(option: string): void {
        if (this.concerns.has(option)) {
            this.concerns.delete(option);
        } else {
            this.concerns.add(option);
        }
    }

    // UE History is served by /data/questionnaire, whose group pages block "Next" until each required
    // question (UEH01-UEH04) has an answer ("You must answer above before submitting.").
    validate(): string | null {
        const questions = [this.handsQuestion, ...this.selfCareQuestions];
        return questions.some(question => !this.answers[question.key])
            ? 'You must answer above before submitting.'
            : null;
    }

    save(): void {
        if (!this.patientId || !this.visitId) {
            this.saveComplete.emit();
            return;
        }

        this.saving = true;
        this.saveError = '';

        this.http.post(`/api/patients/${this.patientId}/ue-history-questionnaire`, {
            visitId: this.visitId,
            handlesObjects: this.answers['handlesObjects'] ?? null,
            dresses: this.answers['dresses'] ?? null,
            bathes: this.answers['bathes'] ?? null,
            toilets: this.answers['toilets'] ?? null,
            armHandConcerns: [...this.concerns]
        }).subscribe({
            next: () => {
                this.saving = false;
                this.saved = true;
                this.cdr.markForCheck();
                this.saveComplete.emit();
            },
            error: error => {
                console.error('Unable to save UE History:', error);
                this.saving = false;
                this.saveError = 'Unable to save. Please try again.';
                this.cdr.markForCheck();
                this.saveFailed.emit(this.saveError);
            }
        });
    }
}














<div class="ue-form">
    @if (saved) {
    <div class="submitted-banner">UE History saved.</div>
    }
    @if (saveError) {
    <div class="questionnaire-error">{{ saveError }}</div>
    }

    <section class="ue-page">
        <h3 class="page-heading">{{ handsQuestion.question }}</h3>
        <div class="radio-column">
            @for (option of handsQuestion.options; track option) {
            <label class="choice-row">
                <input type="radio" [name]="handsQuestion.key" [checked]="answers[handsQuestion.key] === option"
                    (change)="choose(handsQuestion.key, option)" />
                {{ option }}
            </label>
            }
        </div>
    </section>

    <section class="ue-page">
        @for (question of selfCareQuestions; track question.key) {
        <h3 class="page-heading">{{ question.question }}</h3>
        <div class="radio-column">
            @for (option of question.options; track option) {
            <label class="choice-row">
                <input type="radio" [name]="question.key" [checked]="answers[question.key] === option"
                    (change)="choose(question.key, option)" />
                {{ option }}
            </label>
            }
        </div>
        }
    </section>

    <section class="ue-page">
        <h3 class="page-heading">{{ concernsQuestion }}</h3>
        <div class="radio-column">
            @for (option of concernOptions; track option) {
            <label class="choice-row">
                <input type="checkbox" [checked]="concerns.has(option)" (change)="toggleConcern(option)" />
                {{ option }}
            </label>
            }
        </div>
    </section>

    @if (!hideActions) {
    <div class="ue-actions">
        <button type="button" class="submit-btn" [disabled]="saving" (click)="save()">
            {{ saving ? 'Saving...' : (saved ? 'Save Again' : 'Save') }}
        </button>
    </div>
    }
</div>









:host {
    display: block;
}

.ue-form {
    display: flex;
    flex-direction: column;
    gap: 14px;
}

.ue-page {
    display: flex;
    flex-direction: column;
    gap: 12px;
    padding: 18px 20px;
    background: #ffffff;
    border: 1px solid #e2e8f0;
    border-radius: 10px;
}

.page-heading {
    margin: 0;
    color: #1e293b;
    font-size: 15px;
    font-weight: 700;
}

.radio-column {
    display: flex;
    flex-direction: column;
    gap: 2px;
}

.choice-row {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 7px 10px;
    border-radius: 8px;
    font-size: 13px;
    color: #374151;
    cursor: pointer;
}

.choice-row:hover {
    background: #f8fafc;
}

.choice-row input {
    width: 16px;
    height: 16px;
    accent-color: #269c96;
    cursor: pointer;
    flex-shrink: 0;
}

.submitted-banner {
    padding: 12px 16px;
    background: #eef6f4;
    border: 1px solid #a6d8cf;
    border-radius: 8px;
    color: #1f6b5e;
    font-size: 13px;
    font-weight: 500;
}

.questionnaire-error {
    padding: 12px 14px;
    border: 1px solid #efc5c5;
    border-radius: 8px;
    background: #fff4f4;
    color: #b42318;
    font-size: 13px;
}

.ue-actions {
    display: flex;
    justify-content: flex-end;
}

.submit-btn {
    height: 40px;
    padding: 0 22px;
    color: #ffffff;
    background: #269c96;
    border: 1px solid #1f847f;
    border-radius: 8px;
    font-family: inherit;
    font-size: 14px;
    font-weight: 600;
    cursor: pointer;
}

.submit-btn:disabled {
    background: #b7d4d2;
    border-color: #b7d4d2;
    cursor: not-allowed;
}










package org.nemours.gaitlab.controller;

import org.nemours.gaitlab.requests.HipQuestionnaireRequest;
import org.nemours.gaitlab.service.HipQuestionnaireService;
import org.nemours.gaitlab.service.HipQuestionnaireService.HipQuestionnaireResponse;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/patients/{patientId}/hip-questionnaire")
public class HipQuestionnaireController {

    private final HipQuestionnaireService service;

    public HipQuestionnaireController(HipQuestionnaireService service) {
        this.service = service;
    }

    @GetMapping("/{visitId}")
    public HipQuestionnaireResponse find(
            @PathVariable Integer patientId,
            @PathVariable Integer visitId) {
        return service.find(patientId, visitId);
    }

    @PostMapping
    public ResponseEntity<Void> save(
            @PathVariable Integer patientId,
            @RequestBody HipQuestionnaireRequest request) {
        service.save(patientId, request);
        return ResponseEntity.noContent().build();
    }
}












package org.nemours.gaitlab.controller;

import java.util.Map;

import org.nemours.gaitlab.requests.PodciQuestionnaireRequest;
import org.nemours.gaitlab.service.PodciQuestionnaireService;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/patients/{patientId}/podci-questionnaire")
public class PodciQuestionnaireController {

    private final PodciQuestionnaireService service;

    public PodciQuestionnaireController(PodciQuestionnaireService service) {
        this.service = service;
    }

    @GetMapping("/{visitId}")
    public Map<String, Object> find(
            @PathVariable Integer patientId,
            @PathVariable Integer visitId,
            @RequestParam String variant) {
        return service.find(patientId, visitId, variant);
    }

    @PostMapping
    public ResponseEntity<Void> save(
            @PathVariable Integer patientId,
            @RequestBody PodciQuestionnaireRequest request) {
        service.save(patientId, request);
        return ResponseEntity.noContent().build();
    }
}











package org.nemours.gaitlab.controller;

import java.util.List;

import org.nemours.gaitlab.requests.AssembleQuestionnaireRequest;
import org.nemours.gaitlab.service.QuestionnairePickerService;
import org.nemours.gaitlab.service.QuestionnairePickerService.CodeAssignment;

import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/patients/{patientId}/questionnaire-picker")
public class QuestionnairePickerController {

    private final QuestionnairePickerService service;

    public QuestionnairePickerController(QuestionnairePickerService service) {
        this.service = service;
    }

    // Response shape changed from string[] of codes to [{code, visitId}] so each
    // questionnaire can be targeted at its own (possibly retargeted) coincident visit.
    @PostMapping("/assemble")
    public List<CodeAssignment> assemble(
            @PathVariable Integer patientId,
            @RequestBody AssembleQuestionnaireRequest request) {
        return service.assemble(patientId, request);
    }
}











package org.nemours.gaitlab.controller;

import java.time.LocalDate;
import java.util.List;
import java.util.Map;

import org.nemours.gaitlab.model.Visit;
import org.nemours.gaitlab.repository.VisitRepository;
import org.nemours.gaitlab.requests.UpdateQuestionnaireResponseRequest;
import org.nemours.gaitlab.service.QuestionnaireResponseService;

import org.springframework.dao.DataIntegrityViolationException;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;

import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import org.springframework.web.server.ResponseStatusException;

@RestController
@RequestMapping(
        "/api/patients/{patientId}/questionnaire-responses"
)
public class QuestionnaireResponseController {

    private final QuestionnaireResponseService service;
    private final VisitRepository visitRepository;


    public QuestionnaireResponseController(
            QuestionnaireResponseService service,
            VisitRepository visitRepository
    ) {
        this.service = service;
        this.visitRepository = visitRepository;
    }


    @GetMapping
    public Map<String, Object> getPatientResponses(
            @PathVariable int patientId
    ) {

        return service.getPatientResponses(
                patientId
        );
    }


    @GetMapping("/options")
    public Map<String, Object> getOptions(
            @PathVariable int patientId,
            @RequestParam(required = false) String language
    ) {

        return service.getOptions(
                patientId,
                language
        );
    }


    @GetMapping("/visits/{visitId}")
    public Map<String, Object> getVisitResponses(
            @PathVariable int patientId,
            @PathVariable int visitId
    ) {

        return service.getVisitResponses(
                patientId,
                visitId
        );
    }


    @PutMapping("/baseline")
    public ResponseEntity<Void> updateBaselineResponses(
            @PathVariable int patientId,
            @RequestBody
            UpdateQuestionnaireResponseRequest request
    ) {

        service.updateResponses(
                patientId,
                null,
                "baseline",
                request
        );


        return ResponseEntity
                .noContent()
                .build();
    }


    @PutMapping(
            "/visits/{visitId}/{section}"
    )
    public ResponseEntity<Void> updateVisitResponses(
            @PathVariable int patientId,
            @PathVariable int visitId,
            @PathVariable String section,
            @RequestBody
            UpdateQuestionnaireResponseRequest request
    ) {

        service.updateResponses(
                patientId,
                visitId,
                section,
                request
        );


        return ResponseEntity
                .noContent()
                .build();
    }


    @GetMapping("/by-date/{date}")
    public Map<String, Object> getResponsesByDate(
            @PathVariable int patientId,
            @PathVariable LocalDate date
    ) {

        return service.getVisitResponses(
                patientId,
                resolveVisitId(patientId, date)
        );
    }


    @PutMapping(
            "/by-date/{date}/{section}"
    )
    public ResponseEntity<Void> updateResponsesByDate(
            @PathVariable int patientId,
            @PathVariable LocalDate date,
            @PathVariable String section,
            @RequestBody
            UpdateQuestionnaireResponseRequest request
    ) {

        service.updateResponses(
                patientId,
                resolveVisitId(patientId, date),
                section,
                request
        );


        return ResponseEntity
                .noContent()
                .build();
    }


    // Same visit-resolution rule as the Sports/Hip/PODCI save endpoints.
    private int resolveVisitId(int patientId, LocalDate date) {

        List<Visit> visits = visitRepository.findByPatientIdAndDateOrderByIdAsc(patientId, date);

        if (visits.isEmpty()) {
            throw new ResponseStatusException(
                    HttpStatus.BAD_REQUEST,
                    "No visit exists for that date."
            );
        }

        return visits.get(0).getId();
    }


    @ExceptionHandler(
            ResponseStatusException.class
    )
    public ResponseEntity<Map<String, String>>
    handleResponseStatus(
            ResponseStatusException exception
    ) {

        String message =
                exception.getReason() == null
                        ? "Request failed."
                        : exception.getReason();


        return ResponseEntity
                .status(
                        exception.getStatusCode()
                )
                .body(
                        Map.of(
                                "message",
                                message
                        )
                );
    }


    @ExceptionHandler(
            DataIntegrityViolationException.class
    )
    public ResponseEntity<Map<String, String>>
    handleDataIntegrityViolation() {

        return ResponseEntity
                .status(409)
                .body(
                        Map.of(
                                "message",
                                "The database could not save these questionnaire response changes. "
                                +
                                "Close and reopen the editor, then retry."
                        )
                );
    }
}
















package org.nemours.gaitlab.controller;

import java.util.List;

import org.nemours.gaitlab.service.QuestionnairePickerService.CodeAssignment;
import org.nemours.gaitlab.service.QuestionnaireTrackingService;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/patients/{patientId}/questionnaire-tracking")
public class QuestionnaireTrackingController {

    private final QuestionnaireTrackingService service;

    public QuestionnaireTrackingController(QuestionnaireTrackingService service) {
        this.service = service;
    }

    @PostMapping
    public ResponseEntity<Void> track(
            @PathVariable Integer patientId,
            @RequestBody List<CodeAssignment> assignments) {
        service.track(patientId, assignments);
        return ResponseEntity.noContent().build();
    }
}











package org.nemours.gaitlab.controller;

import org.nemours.gaitlab.requests.SportsQuestionnaireRequest;
import org.nemours.gaitlab.service.SportsQuestionnaireService;
import org.nemours.gaitlab.service.SportsQuestionnaireService.SportsQuestionnaireResponse;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/patients/{patientId}/sports-questionnaire")
public class SportsQuestionnaireController {

    private final SportsQuestionnaireService service;

    public SportsQuestionnaireController(SportsQuestionnaireService service) {
        this.service = service;
    }

    @GetMapping("/{visitId}")
    public SportsQuestionnaireResponse find(
            @PathVariable Integer patientId,
            @PathVariable Integer visitId) {
        return service.find(patientId, visitId);
    }

    @PostMapping
    public ResponseEntity<Void> save(
            @PathVariable Integer patientId,
            @RequestBody SportsQuestionnaireRequest request) {
        service.save(patientId, request);
        return ResponseEntity.noContent().build();
    }
}












package org.nemours.gaitlab.controller;

import org.nemours.gaitlab.requests.UeHistoryQuestionnaireRequest;
import org.nemours.gaitlab.service.UeHistoryQuestionnaireService;
import org.nemours.gaitlab.service.UeHistoryQuestionnaireService.UeHistoryQuestionnaireResponse;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/patients/{patientId}/ue-history-questionnaire")
public class UeHistoryQuestionnaireController {

    private final UeHistoryQuestionnaireService service;

    public UeHistoryQuestionnaireController(UeHistoryQuestionnaireService service) {
        this.service = service;
    }

    @GetMapping("/{visitId}")
    public UeHistoryQuestionnaireResponse find(
            @PathVariable Integer patientId,
            @PathVariable Integer visitId) {
        return service.find(patientId, visitId);
    }

    @PostMapping
    public ResponseEntity<Void> save(
            @PathVariable Integer patientId,
            @RequestBody UeHistoryQuestionnaireRequest request) {
        service.save(patientId, request);
        return ResponseEntity.noContent().build();
    }
}










package org.nemours.gaitlab.service;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.stream.IntStream;

import org.nemours.gaitlab.repository.HipQuestionnaireRepository;
import org.nemours.gaitlab.requests.HipQuestionnaireRequest;

import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.web.server.ResponseStatusException;

@Service
public class HipQuestionnaireService {

    private static final List<String> WOMAC_KEYS = IntStream.rangeClosed(1, 24)
        .mapToObj(i -> String.format("womac_%02d", i))
        .toList();

    private static final List<String> HARRIS_KEYS = IntStream.rangeClosed(1, 8)
        .mapToObj(i -> String.format("harris_%02d", i))
        .toList();

    public record HipQuestionnaireResponse(
            Map<String, Integer> womac,
            Map<String, Integer> harris,
            Integer ucla) {}

    private final HipQuestionnaireRepository repository;

    public HipQuestionnaireService(HipQuestionnaireRepository repository) {
        this.repository = repository;
    }

    public HipQuestionnaireResponse find(int patientId, int visitId) {
        Integer visitPatientId = repository.findVisitPatientId(visitId);

        if (visitPatientId == null) {
            throw bad("Visit not found.");
        }

        if (!visitPatientId.equals(patientId)) {
            throw bad("Visit does not belong to this patient.");
        }

        Map<String, Object> row = repository.findByVisitId(visitId);

        if (row == null) {
            return new HipQuestionnaireResponse(Map.of(), Map.of(), null);
        }

        LinkedHashMap<String, Integer> womac = new LinkedHashMap<>();

        for (String key : WOMAC_KEYS) {
            if (row.get(key) != null) {
                womac.put(key, (Integer) row.get(key));
            }
        }

        LinkedHashMap<String, Integer> harris = new LinkedHashMap<>();

        for (String key : HARRIS_KEYS) {
            if (row.get(key) != null) {
                harris.put(key, (Integer) row.get(key));
            }
        }

        Integer ucla = null;

        for (int i = 1; i <= 10; i++) {
            Object value = row.get(String.format("ucla_%02d", i));

            if (value != null && ((Integer) value) == 1) {
                ucla = i;
                break;
            }
        }

        return new HipQuestionnaireResponse(womac, harris, ucla);
    }

    public void save(int patientId, HipQuestionnaireRequest request) {
        if (request.visitId() == null) {
            throw bad("visitId is required.");
        }

        Integer visitPatientId = repository.findVisitPatientId(request.visitId());

        if (visitPatientId == null) {
            throw bad("Visit not found.");
        }

        if (!visitPatientId.equals(patientId)) {
            throw bad("Visit does not belong to this patient.");
        }

        int visitId = request.visitId();

        Map<String, Integer> womac = request.womac() == null ? Map.of() : request.womac();
        Map<String, Integer> harris = request.harris() == null ? Map.of() : request.harris();

        for (String key : womac.keySet()) {
            if (!WOMAC_KEYS.contains(key)) {
                throw bad("Unknown womac field: " + key);
            }
        }

        for (String key : harris.keySet()) {
            if (!HARRIS_KEYS.contains(key)) {
                throw bad("Unknown harris field: " + key);
            }
        }

        if (request.ucla() != null && (request.ucla() < 1 || request.ucla() > 10)) {
            throw bad("ucla must be between 1 and 10.");
        }

        // Partial merge: only columns actually present in this request are written, so
        // submitting one page (e.g. WOMAC) never nulls out answers already saved on another
        // (e.g. Harris) — matches legacy, where each hip-*.jsp page only touches its own fields.
        LinkedHashMap<String, Object> columns = new LinkedHashMap<>();

        for (String key : WOMAC_KEYS) {
            if (!womac.containsKey(key)) {
                continue;
            }

            Integer value = womac.get(key);

            if (value != null && (value < 0 || value > 4)) {
                throw bad("Invalid value for " + key);
            }

            columns.put(key, value);
        }

        for (String key : HARRIS_KEYS) {
            if (harris.containsKey(key)) {
                columns.put(key, harris.get(key));
            }
        }

        // Legacy never writes the rollup columns (ucla, womac_pain/stiffness/function,
        // modified_harris) on save; they are computed on read (HipScore's transient getters).
        if (request.ucla() != null) {
            for (int i = 1; i <= 10; i++) {
                columns.put(String.format("ucla_%02d", i), request.ucla() == i ? 1 : 0);
            }
        }

        if (!columns.isEmpty()) {
            repository.upsert(visitId, columns);
        }
    }

    private static ResponseStatusException bad(String reason) {
        return new ResponseStatusException(HttpStatus.BAD_REQUEST, reason);
    }
}










package org.nemours.gaitlab.service;

import java.util.HashSet;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.Set;

import org.nemours.gaitlab.repository.PodciQuestionnaireRepository;
import org.nemours.gaitlab.requests.PodciQuestionnaireRequest;

import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.web.server.ResponseStatusException;

@Service
public class PodciQuestionnaireService {

    private static final Set<String> VALID_VARIANTS = Set.of("ch", "ap", "as");

    // The 16-condition x 3(a/b/c) grid (q2_007a..q2_022c): legacy radio buttons are
    // value="1" (Yes) / value="2" (No), not a 1/0 checkbox pattern like every other
    // smallint field here.
    private static final Set<String> YES_NO_GRID_FIELDS = buildYesNoGridFields();

    private static Set<String> buildYesNoGridFields() {
        Set<String> fields = new HashSet<>();

        for (int question = 7; question <= 22; question++) {
            for (char suffix : new char[] {'a', 'b', 'c'}) {
                fields.add(String.format("q2_%03d%c", question, suffix));
            }
        }

        return fields;
    }

    private final PodciQuestionnaireRepository repository;

    public PodciQuestionnaireService(PodciQuestionnaireRepository repository) {
        this.repository = repository;
    }

    public Map<String, Object> find(int patientId, int visitId, String variant) {
        if (!VALID_VARIANTS.contains(variant)) {
            throw bad("variant must be one of ch, ap, as.");
        }

        Integer visitPatientId = repository.findVisitPatientId(visitId);

        if (visitPatientId == null) {
            throw bad("Visit not found.");
        }

        if (!visitPatientId.equals(patientId)) {
            throw bad("Visit does not belong to this patient.");
        }

        return repository.findAnswers(visitId, variant);
    }

    public void save(int patientId, PodciQuestionnaireRequest request) {
        if (request.variant() == null || !VALID_VARIANTS.contains(request.variant())) {
            throw bad("variant must be one of ch, ap, as.");
        }

        if (request.visitId() == null) {
            throw bad("visitId is required.");
        }

        Integer visitPatientId = repository.findVisitPatientId(request.visitId());

        if (visitPatientId == null) {
            throw bad("Visit not found.");
        }

        if (!visitPatientId.equals(patientId)) {
            throw bad("Visit does not belong to this patient.");
        }

        int visitId = request.visitId();

        Map<String, Object> answers = request.answers() == null ? Map.of() : request.answers();
        Map<String, String> columnTypes = repository.findAnswerColumnTypes();

        LinkedHashMap<String, Object> columns = new LinkedHashMap<>();

        for (Map.Entry<String, Object> entry : answers.entrySet()) {
            String key = entry.getKey();
            String dbType = columnTypes.get(key);

            if (dbType == null) {
                throw bad("Unknown field: " + key);
            }

            columns.put(key, coerce(key, dbType, entry.getValue()));
        }

        repository.upsert(visitId, request.variant(), columns);
    }

    private static Object coerce(String key, String dbType, Object value) {
        if (value == null) {
            return null;
        }

        if ("smallint".equals(dbType)) {
            if (value instanceof Boolean bool) {
                return bool ? 1 : (YES_NO_GRID_FIELDS.contains(key) ? 2 : 0);
            }

            if (value instanceof Number number) {
                return number.intValue();
            }

            throw bad("Invalid value for " + key);
        }

        if ("boolean".equals(dbType)) {
            if (value instanceof Boolean bool) {
                return bool;
            }

            throw bad("Invalid value for " + key);
        }

        if ("text".equals(dbType)) {
            if (value instanceof String text) {
                return text;
            }

            throw bad("Invalid value for " + key);
        }

        throw bad("Unsupported field: " + key);
    }

    private static ResponseStatusException bad(String reason) {
        return new ResponseStatusException(HttpStatus.BAD_REQUEST, reason);
    }
}












package org.nemours.gaitlab.service;

import java.time.temporal.ChronoUnit;
import java.util.ArrayList;
import java.util.List;

import org.nemours.gaitlab.model.Patient;
import org.nemours.gaitlab.model.Visit;
import org.nemours.gaitlab.repository.PatientRepository;
import org.nemours.gaitlab.repository.VisitRepository;
import org.nemours.gaitlab.requests.AssembleQuestionnaireRequest;

import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.web.server.ResponseStatusException;

@Service
public class QuestionnairePickerService {

    private static final Integer GAIT_VISIT_TYPE = 1;
    private static final Integer UE_VISIT_TYPE = 2;
    private static final Integer RUNNING_VISIT_TYPE = 3;
    private static final Integer RESEARCH_VISIT_TYPE = 4;

    // Matches legacy Visit.knownToBeNotAttended(): attendance == "No" || "Cancel".
    private static final List<String> NOT_ATTENDED = List.of("No", "Cancel");

    public record CodeAssignment(String code, Integer visitId) {}

    private final PatientRepository patientRepository;
    private final VisitRepository visitRepository;

    public QuestionnairePickerService(
            PatientRepository patientRepository,
            VisitRepository visitRepository) {
        this.patientRepository = patientRepository;
        this.visitRepository = visitRepository;
    }

    public List<CodeAssignment> assemble(Integer patientId, AssembleQuestionnaireRequest request) {
        if (request.visitId() == null) {
            throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "visitId is required.");
        }

        Visit referenceVisit = visitRepository.findById(request.visitId())
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.BAD_REQUEST, "Visit not found."));

        if (!referenceVisit.getPatientId().equals(patientId)) {
            throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "Visit does not belong to this patient.");
        }

        Patient patient = patientRepository.findById(patientId)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND, "Patient not found."));

        // Same-day visits, ordered so "last match" (highest id) wins ties, matching
        // legacy ComparisonHelper.getCoincidentVisits' overwrite-in-iteration-order semantics.
        List<Visit> coincidentVisits = visitRepository.findByPatientIdAndDateOrderByIdAsc(patientId, referenceVisit.getDate());

        Visit gaitVisit = lastMatch(coincidentVisits, GAIT_VISIT_TYPE);

        if (referenceVisit.getVisitTypeId() == null) {
            gaitVisit = referenceVisit;
        }

        Visit ueVisit = lastMatch(coincidentVisits, UE_VISIT_TYPE);

        // find_quest.jsp: "Probably best if as much data as possible goes to the Gait visit."
        Visit primaryVisit = (gaitVisit != null && !gaitVisit.getId().equals(referenceVisit.getId()))
                ? gaitVisit
                : referenceVisit;

        List<String> selections = request.selections() == null ? List.of() : request.selections();
        List<CodeAssignment> assignments = new ArrayList<>();

        if (selections.contains("history")) {
            String historyPerson = patient.getHistoryPerson();
            String historyCode = historyPerson == null || historyPerson.isEmpty() ? "NEW_HISTORY" : "HISTORY";
            assignments.add(new CodeAssignment(historyCode, primaryVisit.getId()));

            if (gaitVisit != null) {
                assignments.add(new CodeAssignment("HISTORY_GAIT", gaitVisit.getId()));
            } else if (ueVisit == null && RESEARCH_VISIT_TYPE.equals(referenceVisit.getVisitTypeId())) {
                assignments.add(new CodeAssignment("HISTORY_GAIT", primaryVisit.getId()));
            }

            if (ueVisit != null) {
                assignments.add(new CodeAssignment("UE_HISTORY", ueVisit.getId()));
            }

            assignments.add(new CodeAssignment("HISTORY_CONCERNS", primaryVisit.getId()));
        }

        if (selections.contains("podci-parent")) {
            long age = ChronoUnit.DAYS.between(patient.getDateOfBirth(), primaryVisit.getDate()) / 365;

            if (age >= 11) {
                assignments.add(new CodeAssignment(
                        "sp".equals(request.language()) ? "PODCI_CH" : "PODCI_AP", primaryVisit.getId()));
            } else {
                assignments.add(new CodeAssignment("PODCI_CH", primaryVisit.getId()));
            }
        }

        if (selections.contains("sports")) {
            Visit runningVisit = lastAttendedMatch(coincidentVisits, RUNNING_VISIT_TYPE);
            assignments.add(new CodeAssignment(
                    "SPORTS", runningVisit != null ? runningVisit.getId() : primaryVisit.getId()));
        }

        if (selections.contains("hip")) {
            assignments.add(new CodeAssignment("HIP", primaryVisit.getId()));
        }

        if (selections.contains("podci-self")) {
            assignments.add(new CodeAssignment("PODCI_AS", primaryVisit.getId()));
        }

        // Stable partition: self-answered PODCI (PODCI_AS) sorted last, everything else keeps its order.
        assignments.sort((a, b) -> Boolean.compare("PODCI_AS".equals(a.code()), "PODCI_AS".equals(b.code())));

        return assignments;
    }

    private static Visit lastMatch(List<Visit> visits, Integer visitTypeId) {
        Visit match = null;

        for (Visit visit : visits) {
            if (visitTypeId.equals(visit.getVisitTypeId())) {
                match = visit;
            }
        }

        return match;
    }

    private static Visit lastAttendedMatch(List<Visit> visits, Integer visitTypeId) {
        Visit match = null;

        for (Visit visit : visits) {
            if (visitTypeId.equals(visit.getVisitTypeId()) && !NOT_ATTENDED.contains(visit.getAttendance())) {
                match = visit;
            }
        }

        return match;
    }
}











package org.nemours.gaitlab.service;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;

import java.nio.charset.StandardCharsets;

import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;

import java.time.LocalDate;
import java.time.format.DateTimeParseException;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.HashMap;
import java.util.HashSet;
import java.util.HexFormat;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.Set;
import java.util.stream.Collectors;

import org.nemours.gaitlab.repository.QuestionnaireResponseRepository;
import org.nemours.gaitlab.requests.UpdateQuestionnaireResponseRequest;

import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Isolation;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.server.ResponseStatusException;

@Service
public class QuestionnaireResponseService {

    private final QuestionnaireResponseRepository repository;
    private final ObjectMapper json;

    public QuestionnaireResponseService(
            QuestionnaireResponseRepository repository,
            ObjectMapper json) {
        this.repository = repository;
        this.json = json;
    }
    public record QuestionnaireResponseSection(
            Map<String, Object> fields,
            Map<String, List<Map<String, Object>>> lists,
            String version) {
    }

    private record ListSpec(
            String table,
            LinkedHashMap<String, String> columns) {
    }

    private static LinkedHashMap<String, String> columns(
            String... pairs) {

        LinkedHashMap<String, String> result = new LinkedHashMap<>();

        for (int i = 0; i < pairs.length; i += 2) {

            result.put(
                    pairs[i],
                    pairs[i + 1]);
        }

        return result;
    }

    // =========================================================
    // BASELINE FIELDS -> patients
    // =========================================================

    private static final LinkedHashMap<String, String> BASELINE = columns(

            "historyPerson",
            "hist_person",

            "historyRelationship",
            "hist_rship",

            "baselineDate",
            "date_baseline",

            "premature",
            "premature",

            "birthStay",
            "birth_stay",

            "ageWalk",
            "age_walk",

            "ageTalk",
            "age_talk",

            "learning",
            "learning");

    // =========================================================
    // VISIT QUESTIONNAIRE ANSWERS -> visits
    // =========================================================

    private static final LinkedHashMap<String, String> ITEMS = columns(

            "reportingPerson",
            "status_name",

            "relationship",
            "status_rship",

            "walkingChange",
            "walk_change",

            "walkingSupportCode",
            "walk_support",

            "fms5Id",
            "quest_fms5",

            "fms50Id",
            "quest_fms50",

            "fms500Id",
            "quest_fms500");

    // =========================================================
    // QUESTIONNAIRE DETAILS -> visits
    // =========================================================

    private static final LinkedHashMap<String, String> DETAILS = columns(

            "hospitalPt",
            "hosp_pt",

            "clinicPt",
            "clinic_pt",

            "schoolPt",
            "school_pt",

            "homePt",
            "home_pt",

            "hospitalOt",
            "hosp_ot",

            "clinicOt",
            "clinic_ot",

            "schoolOt",
            "school_ot",

            "homeOt",
            "home_ot",

            "followup",
            "followup",

            "generalConcerns",
            "concerns");

    private static final Set<String> NUMBERS = Set.of(
            "learning",
            "fms5Id",
            "fms50Id",
            "fms500Id");

    // =========================================================
    // REPEATING LIST STORAGE
    // =========================================================

    private static final Map<String, ListSpec> LISTS = Map.of(

            "medications",
            new ListSpec(
                    "pt_seizure_meds",
                    columns(
                            "name",
                            "seizure_med")),

            "devices",
            new ListSpec(
                    "pt_devices",
                    columns(
                            "name",
                            "device",

                            "side",
                            "side")),

            "gaitConcerns",
            new ListSpec(
                    "pt_gait_concerns",
                    columns(
                            "code",
                            "concern")),

            "pain",
            new ListSpec(
                    "pt_pain",
                    columns(
                            "bodyPart",
                            "part",

                            "side",
                            "side")));

    private static final List<Map<String, Object>> WALK_CHANGES = List.of(

            Map.of("value", "No Change", "label", "No Change"),
            Map.of("value", "Much Better", "label", "Walks Much Better"),
            Map.of("value", "Little Better", "label", "Walks a Little Better"),
            Map.of("value", "Little Worse", "label", "Walks a Little Worse"),
            Map.of("value", "Much Worse", "label", "Walks Much Worse"));

    private static final Set<String> WALK_CHANGE_VALUES = WALK_CHANGES.stream()
            .map(option -> (String) option.get("value"))
            .collect(Collectors.toSet());

    private static LinkedHashMap<String, String> fields(
            String section) {

        return switch (section) {

            case "baseline" ->
                BASELINE;

            case "items" ->
                ITEMS;

            case "details" ->
                DETAILS;

            case "medications" ->
                new LinkedHashMap<>();

            default ->
                throw bad(
                        "Unknown questionnaire response section.");
        };
    }

    private static List<String> listNames(
            String section) {

        return switch (section) {

            case "medications" ->
                List.of(
                        "medications");

            case "details" ->
                List.of(
                        "devices",
                        "gaitConcerns",
                        "pain");

            default ->
                List.of();
        };
    }

    // =========================================================
    // BUILD ONE SECTION
    // =========================================================

    private QuestionnaireResponseSection section(
            int patientId,
            Integer visitId,
            String name) {

        Map<String, Object> values = new LinkedHashMap<>();

        LinkedHashMap<String, String> mapping = fields(name);

        if (!mapping.isEmpty()) {

            values = name.equals("baseline")

                    ? repository.findPatientFields(
                            patientId,
                            mapping)

                    : repository.findVisitFields(
                            patientId,
                            visitId,
                            mapping);
        }

        Map<String, List<Map<String, Object>>> lists = new LinkedHashMap<>();

        for (String listName : listNames(name)) {

            ListSpec spec = LISTS.get(listName);

            lists.put(
                    listName,
                    repository.findListRows(
                            visitId,
                            spec.table(),
                            spec.columns()));
        }

        return new QuestionnaireResponseSection(
                values,
                lists,
                version(
                        Arrays.asList(
                                patientId,
                                visitId,
                                name,
                                values,
                                lists)));
    }

    // =========================================================
    // VERSION FOR STALE-EDIT PROTECTION
    // =========================================================

    private String version(
            Object value) {

        try {

            byte[] bytes = json.writeValueAsString(value)
                    .getBytes(
                            StandardCharsets.UTF_8);

            return HexFormat
                    .of()
                    .formatHex(

                            MessageDigest
                                    .getInstance(
                                            "SHA-256")
                                    .digest(bytes));

        } catch (
                JsonProcessingException | NoSuchAlgorithmException exception) {

            throw new IllegalStateException(
                    "Unable to version questionnaire responses.",
                    exception);
        }
    }

    // =========================================================
    // MAIN RESPONSE PAGE
    // =========================================================

    @Transactional(readOnly = true, isolation = Isolation.REPEATABLE_READ)
    public Map<String, Object> getPatientResponses(
            int patientId) {

        repository.requirePatient(
                patientId,
                false);

        Map<String, Object> response = new LinkedHashMap<>();

        response.put(
                "baseline",
                section(
                        patientId,
                        null,
                        "baseline"));

        response.put(
                "visits",
                repository.findEligibleVisits(
                        patientId));

        return response;
    }

    // =========================================================
    // SELECTED VISIT
    // =========================================================

    @Transactional(readOnly = true, isolation = Isolation.REPEATABLE_READ)
    public Map<String, Object> getVisitResponses(
            int patientId,
            int visitId) {

        Map<String, Object> selectedVisit = repository.findOwnedVisit(
                patientId,
                visitId,
                false);

        LinkedHashMap<String, String> allScalarFields = new LinkedHashMap<>(ITEMS);

        allScalarFields.putAll(
                DETAILS);

        /*
         * Legacy behavior:
         * selected visit first,
         * then other visits on the same date.
         */
        List<Map<String, Object>> sameDayAnswers = repository.findSameDayAnswers(
                patientId,
                visitId,
                allScalarFields);

        Map<String, Object> effective = new LinkedHashMap<>();

        for (String key : allScalarFields.keySet()) {

            effective.put(
                    key,
                    firstAnswer(
                            sameDayAnswers,
                            key));
        }

        Map<String, Object> response = new LinkedHashMap<>();

        response.put(
                "visitId",
                visitId);

        response.put(
                "visitDate",
                selectedVisit.get("date"));

        response.put(
                "effective",
                effective);

        response.put(
                "items",
                section(
                        patientId,
                        visitId,
                        "items"));

        response.put(
                "medications",
                section(
                        patientId,
                        visitId,
                        "medications"));

        response.put(
                "details",
                section(
                        patientId,
                        visitId,
                        "details"));

        return response;
    }

    private static Object firstAnswer(
            List<Map<String, Object>> rows,
            String field) {

        return rows
                .stream()
                .map(
                        row -> row.get(field))
                .filter(
                        value -> value != null
                                &&
                                !"".equals(value))
                .findFirst()
                .orElse(null);
    }

    // =========================================================
    // DROPDOWN OPTIONS
    // =========================================================

    @Transactional(readOnly = true)
    public Map<String, Object> getOptions(
            int patientId,
            String language) {

        repository.requirePatient(
                patientId,
                false);

        if (language != null && !Set.of("en", "sp").contains(language)) {
            throw bad("language must be en or sp.");
        }

        Map<String, Object> result = new LinkedHashMap<>();

        result.put(
                "relationships",
                language == null ? repository.findRelationships() : repository.findRelationships(language));

        result.put(
                "premature",
                language == null ? repository.findPrematureOptions() : repository.findPrematureOptions(language));

        result.put(
                "birthStays",
                language == null ? repository.findBirthStayOptions() : repository.findBirthStayOptions(language));

        result.put(
                "ages",
                language == null ? repository.findAgeOptions() : repository.findAgeOptions(language));

        result.put(
                "learning",
                language == null ? repository.findLearningOptions() : repository.findLearningOptions(language));

        result.put(
                "walkingSupport",
                language == null ? repository.findWalkingSupportOptions() : repository.findWalkingSupportOptions(language));

        result.put(
                "walkingChanges",
                language == null ? WALK_CHANGES : localizedWalkChanges(language));

        result.put(
                "fms",
                language == null ? repository.findFmsOptions() : repository.findFmsOptions(language));

        result.put(
                "medications",
                language == null ? repository.findMedicationOptions() : repository.findMedicationOptions(language));

        result.put(
                "devices",
                language == null ? repository.findDeviceOptions() : repository.findDeviceOptions(language));

        result.put(
                "gaitConcerns",
                language == null ? repository.findGaitConcernOptions() : repository.findGaitConcernOptions(language));

        result.put(
                "pain",
                language == null ? repository.findPainOptions() : repository.findPainOptions(language));

        return result;
    }

    // English labels are the hardcoded WALK_CHANGES text; Spanish is not yet supplied (legacy text
    // pending from q-walking_change.jsp) so it comes back null rather than an invented translation.
    private static List<Map<String, Object>> localizedWalkChanges(String language) {
        List<Map<String, Object>> result = new ArrayList<>();

        for (Map<String, Object> option : WALK_CHANGES) {
            Map<String, Object> copy = new LinkedHashMap<>();
            copy.put("value", option.get("value"));
            copy.put("label", "en".equals(language) ? option.get("label") : null);
            result.add(copy);
        }

        return result;
    }

    // =========================================================
    // SAVE RESPONSE SECTION
    // =========================================================

    @Transactional
    public void updateResponses(
            int patientId,
            Integer visitId,
            String sectionName,
            UpdateQuestionnaireResponseRequest request) {

        LinkedHashMap<String, String> mapping = fields(sectionName);

        if (visitId != null
                &&
                sectionName.equals("baseline")) {

            throw bad(
                    "Use the baseline questionnaire response endpoint.");
        }

        if (sectionName.equals("baseline")) {

            repository.requirePatient(
                    patientId,
                    true);

        } else {

            if (visitId == null) {

                throw bad(
                        "A visit is required.");
            }

            repository.findOwnedVisit(
                    patientId,
                    visitId,
                    true);
        }

        QuestionnaireResponseSection current = section(
                patientId,
                visitId,
                sectionName);

        if (request == null
                ||
                request.version() == null
                ||
                !request.version()
                        .equals(
                                current.version())) {

            throw new ResponseStatusException(
                    HttpStatus.CONFLICT,

                    "These questionnaire responses changed after this editor was opened. "
                            +
                            "Close and reopen the editor to load the latest responses.");
        }

        if (request.fields() == null
                ||
                !request.fields()
                        .keySet()
                        .equals(
                                mapping.keySet())
                ||
                request.lists() == null
                ||
                !request.lists()
                        .keySet()
                        .equals(
                                new HashSet<>(
                                        listNames(sectionName)))) {

            throw bad(
                    "The complete questionnaire response section is required. "
                            +
                            "Reload the page and retry.");
        }

        LinkedHashMap<String, Object> values = new LinkedHashMap<>();

        for (String key : mapping.keySet()) {

            Object value = normalize(
                    key,
                    request
                            .fields()
                            .get(key));

            validateField(
                    key,
                    value,
                    current
                            .fields()
                            .get(key));

            values.put(
                    key,
                    value);
        }

        /*
         * Validate every list before making database writes.
         */
        for (String listName : listNames(sectionName)) {

            validateRows(
                    listName,

                    request
                            .lists()
                            .get(listName),

                    current
                            .lists()
                            .get(listName));
        }

        if (!mapping.isEmpty()) {

            if (sectionName.equals("baseline")) {

                repository.updatePatientFields(
                        patientId,
                        mapping,
                        values);

            } else {

                repository.updateVisitFields(
                        patientId,
                        visitId,
                        mapping,
                        values);
            }
        }

        for (String listName : listNames(sectionName)) {

            syncRows(
                    visitId,

                    LISTS.get(listName),

                    request
                            .lists()
                            .get(listName),

                    current
                            .lists()
                            .get(listName));
        }
    }

    // =========================================================
    // NORMALIZATION
    // =========================================================

    private static Object normalize(
            String key,
            Object value) {

        if (value == null) {
            return null;
        }

        if (NUMBERS.contains(key)) {

            // edit_baseline.jsp's MM_columnsStr writes learning|none,none,NULL: a blank
            // dropdown selection saves as SQL NULL, not "". These are all optional FK
            // columns (patients.learning, visits.quest_fms5/50/500), so an unselected
            // value means "no answer" here too.
            if ("".equals(value)) {
                return null;
            }

            if (!(value instanceof Number number)
                    ||
                    number.doubleValue() != number.intValue()) {

                throw bad(
                        "Invalid numeric option: " +
                                key);
            }

            return number.intValue();
        }

        if (!(value instanceof String)) {

            throw bad(
                    "Invalid text value: " +
                            key);
        }

        return value;
    }

    // =========================================================
    // FIELD VALIDATION
    // =========================================================

    private void validateField(
            String key,
            Object value,
            Object oldValue) {

        if (key.equals("baselineDate")
                &&
                value != null
                &&
                !"".equals(value)) {

            try {

                LocalDate.parse(
                        (String) value);

            } catch (DateTimeParseException exception) {

                throw bad(
                        "Enter a valid baseline date (YYYY-MM-DD).");
            }
        }

        /*
         * Preserve old legacy values even when
         * they are no longer in the lookup table.
         */
        if (value == null
                ||
                "".equals(value)
                ||
                Objects.equals(
                        value,
                        oldValue)) {

            return;
        }

        if (!repository.validLookup(
                key,
                value)) {

            throw bad(
                    "Invalid selection for " +
                            key +
                            ".");
        }

        if (key.equals("walkingChange")
                &&
                !WALK_CHANGE_VALUES.contains(value)) {

            throw bad(
                    "Invalid walking change.");
        }

        if ((key.endsWith("Pt")
                ||
                key.endsWith("Ot"))
                &&
                !List.of(
                        "1",
                        "2",
                        "3",
                        "4").contains(value)) {

            throw bad(
                    "Invalid therapy frequency.");
        }
    }

    // =========================================================
    // REPEATING LIST VALIDATION
    // =========================================================

    private void validateRows(
            String name,
            List<Map<String, Object>> rows,
            List<Map<String, Object>> oldRows) {

        if (rows == null
                ||
                rows.size() > Math.max(
                        500,
                        oldRows.size())) {

            throw bad(
                    "Invalid list of " +
                            name +
                            ".");
        }

        ListSpec spec = LISTS.get(name);

        Set<String> expected = new HashSet<>(
                spec
                        .columns()
                        .keySet());

        expected.add("id");

        Map<Integer, Map<String, Object>> previous = new HashMap<>();

        for (Map<String, Object> old : oldRows) {

            previous.put(
                    ((Number) old.get("id"))
                            .intValue(),
                    old);
        }

        Set<Integer> seen = new HashSet<>();

        for (Map<String, Object> row : rows) {

            if (row == null
                    ||
                    !row.keySet()
                            .equals(expected)) {

                throw bad(
                        "Invalid row in " +
                                name +
                                ".");
            }

            Integer id = rowId(
                    row.get("id"));

            if (id != null
                    &&
                    (!previous.containsKey(id)
                            ||
                            !seen.add(id))) {

                throw bad(
                        "A row does not belong to this visit, "
                                +
                                "or is repeated.");
            }

            Map<String, Object> old = id == null
                    ? Map.of()
                    : previous.get(id);

            for (String key : spec
                    .columns()
                    .keySet()) {

                normalize(
                        key,
                        row.get(key));
            }

            String primary = spec
                    .columns()
                    .keySet()
                    .iterator()
                    .next();

            Object value = row.get(primary);

            if (Objects.equals(
                    value,
                    old.get(primary))
                    &&
                    id != null) {

                continue;
            }

            if (!(value instanceof String text)
                    ||
                    text.isBlank()) {

                throw bad(
                        "Complete or remove the empty row in "
                                +
                                name +
                                ".");
            }

            if (name.equals("gaitConcerns")
                    &&
                    !repository.gaitConcernExists(
                            value)) {

                throw bad(
                        "Invalid gait concern.");
            }

            /*
             * Medication, device, and pain text remain
             * open text because the legacy application
             * allows custom values.
             */
        }
    }

    private static Integer rowId(
            Object value) {

        if (value == null) {
            return null;
        }

        if (!(value instanceof Number number)
                ||
                number.doubleValue() != number.intValue()) {

            throw bad(
                    "Invalid row ID.");
        }

        return number.intValue();
    }

    // =========================================================
    // REPEATING LIST SAVE
    // =========================================================

    private void syncRows(
            int visitId,
            ListSpec spec,
            List<Map<String, Object>> rows,
            List<Map<String, Object>> previous) {

        Set<Integer> retained = new HashSet<>();

        for (Map<String, Object> row : rows) {

            Integer id = rowId(
                    row.get("id"));

            Map<String, Object> saveRow = normalizeListRow(
                    spec,
                    row);

            if (id == null) {

                repository.insertListRow(
                        visitId,
                        spec.table(),
                        spec.columns(),
                        saveRow);

            } else {

                retained.add(id);

                repository.updateListRow(
                        visitId,
                        id,
                        spec.table(),
                        spec.columns(),
                        saveRow);
            }
        }

        for (Map<String, Object> old : previous) {

            int id = ((Number) old.get("id"))
                    .intValue();

            if (!retained.contains(id)) {

                repository.deleteListRow(
                        visitId,
                        id,
                        spec.table());
            }
        }
    }

    // q-devices.jsp's insert branches on side.equals(""): a blank side always saves as SQL
    // NULL, not "". (Only Crutch/Cane ever show a side picker in the legacy UI; every other
    // device submits a blank hidden field, so this converts every non-Crutch/Cane row's side
    // to NULL too, matching legacy exactly rather than special-casing the device name here.)
    private static Map<String, Object> normalizeListRow(
            ListSpec spec,
            Map<String, Object> row) {

        if (!"pt_devices".equals(spec.table())
                ||
                !"".equals(row.get("side"))) {

            return row;
        }

        Map<String, Object> normalized = new LinkedHashMap<>(row);
        normalized.put("side", null);
        return normalized;
    }

    private static ResponseStatusException bad(
            String reason) {

        return new ResponseStatusException(
                HttpStatus.BAD_REQUEST,
                reason);
    }
}












package org.nemours.gaitlab.service;

import java.util.List;
import java.util.Map;
import java.util.stream.IntStream;

import org.nemours.gaitlab.repository.QuestionnaireTrackingRepository;
import org.nemours.gaitlab.repository.SportsQuestionnaireRepository;
import org.nemours.gaitlab.repository.UeHistoryQuestionnaireRepository;
import org.nemours.gaitlab.repository.VisitRepository;
import org.nemours.gaitlab.service.QuestionnairePickerService.CodeAssignment;

import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.server.ResponseStatusException;

// Ports QuestionnaireServiceImpl.setDescriptor: legacy inserts one q_track row per wizard page
// reached. We save a whole questionnaire in one call, so instead of tracking per-page navigation
// we insert the full legacy page set for each assembled code in a single call after save-all
// succeeds. wasAsked() (the only consumer of this data — comp_report_view.jsp, include_history.jsp)
// pattern-matches "%:<pageName>" and never looks at the index prefix, so the index is cosmetic;
// we use the assembled-list position per the agreed design (cheap, closer to legacy than a constant).
@Service
public class QuestionnaireTrackingService {

    private static final Map<String, List<String>> STATIC_PAGES = Map.ofEntries(
        Map.entry("HISTORY", List.of(
            "history_who", "history_history", "history_conditions", "history_seizures",
            "history_botox", "history_devices", "history_pain", "history_pt", "history_followup")),
        Map.entry("NEW_HISTORY", List.of(
            "new_history_who", "new_history_birth", "new_history_walking", "new_history_learning",
            "history_history", "history_conditions", "history_seizures", "history_botox",
            "history_devices", "history_pain", "history_pt", "history_followup")),
        Map.entry("HISTORY_GAIT", List.of(
            "history_walking_support", "history_fms5", "history_fms50", "history_fms500",
            "history_walking_change", "history_gait_concerns")),
        Map.entry("HISTORY_CONCERNS", List.of("history_concerns")),
        Map.entry("HIP", List.of("hip_womac", "hip_ucla", "hip_harris")));

    // Legacy's UE_HISTORY_ID Struts group chain is hardcoded to length 3 in QuestionnaireServiceImpl
    // (UEH03/04/06 are orphaned, matching what we built).
    private static final List<String> UE_HISTORY_GROUPS = List.of("UEH01", "UEH02", "UEH05");

    // Legacy's SPORTS_ID Struts chain, confirmed linear/no-branching in 01-sports.txt.
    private static final List<String> SPORTS_GROUPS = List.of(
        "SPORTS9", "SPORTS1", "SPORTS2", "SPORTS3", "SPORTS4", "SPORTS5", "SPORTS6", "SPORTS7", "SPORTS8");

    private final QuestionnaireTrackingRepository repository;
    private final VisitRepository visitRepository;
    private final SportsQuestionnaireRepository sportsRepository;
    private final UeHistoryQuestionnaireRepository ueHistoryRepository;

    public QuestionnaireTrackingService(
            QuestionnaireTrackingRepository repository,
            VisitRepository visitRepository,
            SportsQuestionnaireRepository sportsRepository,
            UeHistoryQuestionnaireRepository ueHistoryRepository) {
        this.repository = repository;
        this.visitRepository = visitRepository;
        this.sportsRepository = sportsRepository;
        this.ueHistoryRepository = ueHistoryRepository;
    }

    @Transactional
    public void track(int patientId, List<CodeAssignment> assignments) {
        if (assignments == null || assignments.isEmpty()) {
            throw bad("At least one assigned questionnaire code is required.");
        }

        for (CodeAssignment assignment : assignments) {
            if (assignment.visitId() == null) {
                throw bad("visitId is required for every code.");
            }

            var visit = visitRepository.findById(assignment.visitId())
                .orElseThrow(() -> bad("Visit not found."));

            if (!visit.getPatientId().equals(patientId)) {
                throw bad("Visit does not belong to this patient.");
            }
        }

        IntStream.range(0, assignments.size()).forEach(index -> track(assignments.get(index), index));
    }

    private void track(CodeAssignment assignment, int index) {
        String code = assignment.code();
        int visitId = assignment.visitId();

        if (STATIC_PAGES.containsKey(code)) {
            for (String page : STATIC_PAGES.get(code)) {
                repository.insertIfAbsent(visitId, index + ":" + page, null);
            }
            return;
        }

        if (code.startsWith("PODCI_")) {
            String suffix = code.substring("PODCI_".length()).toLowerCase();

            for (int question = 1; question <= 27; question++) {
                String page = String.format("q2_%s_%03d", suffix, question);
                repository.insertIfAbsent(visitId, index + ":" + page, null);
            }
            return;
        }

        if ("SPORTS".equals(code)) {
            Integer sessionId = sportsRepository.findExistingSessionId(visitId);

            for (String group : SPORTS_GROUPS) {
                repository.insertIfAbsent(visitId, index + ":" + group, sessionId);
            }
            return;
        }

        if ("UE_HISTORY".equals(code)) {
            Integer sessionId = ueHistoryRepository.findExistingSessionId(visitId);

            for (String group : UE_HISTORY_GROUPS) {
                repository.insertIfAbsent(visitId, index + ":" + group, sessionId);
            }
            return;
        }

        // PODCI_AS/CH/AP fall through the "PODCI_" prefix check above; anything else
        // (e.g. a future code we haven't wired up) is silently skipped rather than failing
        // the whole tracking call for the other, valid codes in the same request.
    }

    private static ResponseStatusException bad(String reason) {
        return new ResponseStatusException(HttpStatus.BAD_REQUEST, reason);
    }
}













package org.nemours.gaitlab.service;

import java.util.List;
import java.util.Map;

import org.nemours.gaitlab.repository.SportsQuestionnaireRepository;
import org.nemours.gaitlab.requests.SportsQuestionnaireRequest;

import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.server.ResponseStatusException;

@Service
public class SportsQuestionnaireService {

    private static final int Q_PHYSICAL_THERAPY = 1;
    private static final int Q_COMPETITIVE_SPORTS = 2;
    private static final int Q_YEARS_IN_SPORT = 3;
    private static final int Q_DAYS_PER_WEEK = 4;
    private static final int Q_ORTHOTICS = 5;
    private static final int Q_PAIN_LOCATIONS = 6;
    private static final int Q_MILE_TIME = 7;
    private static final int Q_GOALS = 8;
    private static final int Q_REPORTING_PERSON = 60;

    private final SportsQuestionnaireRepository repository;

    public SportsQuestionnaireService(SportsQuestionnaireRepository repository) {
        this.repository = repository;
    }

    @Transactional
    public void save(int patientId, SportsQuestionnaireRequest request) {
        if (request.visitId() == null) {
            throw bad("visitId is required.");
        }

        Integer visitPatientId = repository.findVisitPatientId(request.visitId());

        if (visitPatientId == null) {
            throw bad("Visit not found.");
        }

        if (!visitPatientId.equals(patientId)) {
            throw bad("Visit does not belong to this patient.");
        }

        int sessionId = repository.findOrCreateSession(request.visitId());

        saveFreeText(sessionId, Q_REPORTING_PERSON, request.reportingPerson());
        saveChoice(sessionId, Q_PHYSICAL_THERAPY, request.physicalTherapy());
        saveFreeText(sessionId, Q_COMPETITIVE_SPORTS, request.competitiveSports());
        saveChoice(sessionId, Q_YEARS_IN_SPORT, request.yearsInSport());
        saveChoice(sessionId, Q_DAYS_PER_WEEK, request.daysPerWeek());
        saveChoice(sessionId, Q_ORTHOTICS, request.orthotics());
        saveMultiChoice(sessionId, Q_PAIN_LOCATIONS, request.painLocations());
        saveMileTime(sessionId, request.mileTimeKnown(), request.mileTime());
        saveFreeText(sessionId, Q_GOALS, request.goals());
    }

    public record SportsQuestionnaireResponse(
            String reportingPerson,
            String physicalTherapy,
            String competitiveSports,
            String yearsInSport,
            String daysPerWeek,
            String orthotics,
            List<String> painLocations,
            String mileTimeKnown,
            String mileTime,
            String goals) {}

    // No "populate=last" fallback to a prior visit: the DB column that would drive that
    // (nmquestionnaire.question.populate) is null live, and the only code that interprets a
    // "last" value (QuestionServiceImpl.getShouldPopulateMostRecentVisit) is never called from
    // anywhere in the legacy source available to us. Returns this visit's own saved answers only.
    public SportsQuestionnaireResponse find(int patientId, int visitId) {
        Integer visitPatientId = repository.findVisitPatientId(visitId);

        if (visitPatientId == null) {
            throw bad("Visit not found.");
        }

        if (!visitPatientId.equals(patientId)) {
            throw bad("Visit does not belong to this patient.");
        }

        Integer sessionId = repository.findExistingSessionId(visitId);

        if (sessionId == null) {
            return new SportsQuestionnaireResponse(
                null, null, null, null, null, null, List.of(), null, null, null);
        }

        Map<String, Object> mileTimeAnswer = repository.findActiveChoiceWithText(sessionId, Q_MILE_TIME);
        String mileTimeKnown = null;
        String mileTime = null;

        if (mileTimeAnswer != null) {
            String label = (String) mileTimeAnswer.get("label");

            if ("Unknown".equals(label)) {
                mileTimeKnown = "unknown";
            } else if ("Time".equals(label)) {
                mileTimeKnown = "known";
                mileTime = (String) mileTimeAnswer.get("free_text");
            }
        }

        return new SportsQuestionnaireResponse(
            repository.findActiveFreeText(sessionId, Q_REPORTING_PERSON),
            firstOrNull(repository.findActiveAnswerLabels(sessionId, Q_PHYSICAL_THERAPY)),
            repository.findActiveFreeText(sessionId, Q_COMPETITIVE_SPORTS),
            firstOrNull(repository.findActiveAnswerLabels(sessionId, Q_YEARS_IN_SPORT)),
            firstOrNull(repository.findActiveAnswerLabels(sessionId, Q_DAYS_PER_WEEK)),
            firstOrNull(repository.findActiveAnswerLabels(sessionId, Q_ORTHOTICS)),
            repository.findActiveAnswerLabels(sessionId, Q_PAIN_LOCATIONS),
            mileTimeKnown,
            mileTime,
            repository.findActiveFreeText(sessionId, Q_GOALS));
    }

    private static String firstOrNull(List<String> values) {
        return values.isEmpty() ? null : values.get(0);
    }

    private void saveFreeText(int sessionId, int questionId, String value) {
        if (value == null || value.isBlank()) {
            return;
        }

        // Matches legacy: free-text answers still point at a real answer_id (its
        // TextAnswerContainer placeholder), not null.
        Integer answerId = repository.findTextAnswerId(questionId);

        repository.deactivateActiveAnswers(sessionId, questionId);
        repository.insertAnswer(sessionId, questionId, answerId, value);
    }

    private void saveChoice(int sessionId, int questionId, String value) {
        if (value == null || value.isBlank()) {
            return;
        }

        Integer answerId = repository.resolveAnswerId(questionId, value);

        if (answerId == null) {
            throw bad("Could not resolve answer for question " + questionId + ": " + value);
        }

        repository.deactivateActiveAnswers(sessionId, questionId);
        repository.insertAnswer(sessionId, questionId, answerId, null);
    }

    private void saveMultiChoice(int sessionId, int questionId, List<String> values) {
        if (values == null || values.isEmpty()) {
            return;
        }

        repository.deactivateActiveAnswers(sessionId, questionId);

        for (String value : values) {
            Integer answerId = repository.resolveAnswerId(questionId, value);

            if (answerId == null) {
                throw bad("Could not resolve answer for question " + questionId + ": " + value);
            }

            repository.insertAnswer(sessionId, questionId, answerId, null);
        }
    }

    private void saveMileTime(int sessionId, String mileTimeKnown, String mileTime) {
        if (mileTimeKnown == null || mileTimeKnown.isBlank()) {
            return;
        }

        String text;

        if ("unknown".equals(mileTimeKnown)) {
            text = "Unknown";
        } else if ("known".equals(mileTimeKnown)) {
            text = "Time";
        } else {
            throw bad("mileTimeKnown must be 'unknown' or 'known'.");
        }

        Integer answerId = repository.resolveAnswerId(Q_MILE_TIME, text);

        if (answerId == null) {
            throw bad("Could not resolve mile time answer: " + text);
        }

        String answerText = "known".equals(mileTimeKnown) ? mileTime : null;

        repository.deactivateActiveAnswers(sessionId, Q_MILE_TIME);
        repository.insertAnswer(sessionId, Q_MILE_TIME, answerId, answerText);
    }

    private static ResponseStatusException bad(String reason) {
        return new ResponseStatusException(HttpStatus.BAD_REQUEST, reason);
    }
}











package org.nemours.gaitlab.service;

import java.util.List;

import org.nemours.gaitlab.repository.UeHistoryQuestionnaireRepository;
import org.nemours.gaitlab.requests.UeHistoryQuestionnaireRequest;

import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.server.ResponseStatusException;

@Service
public class UeHistoryQuestionnaireService {

    private static final int Q_HANDLES_OBJECTS = 54;
    private static final int Q_DRESSES = 55;
    private static final int Q_BATHES = 56;
    private static final int Q_TOILETS = 57;
    private static final int Q_ARM_HAND_CONCERNS = 58;

    private final UeHistoryQuestionnaireRepository repository;

    public UeHistoryQuestionnaireService(UeHistoryQuestionnaireRepository repository) {
        this.repository = repository;
    }

    @Transactional
    public void save(int patientId, UeHistoryQuestionnaireRequest request) {
        if (request.visitId() == null) {
            throw bad("visitId is required.");
        }

        Integer visitPatientId = repository.findVisitPatientId(request.visitId());

        if (visitPatientId == null) {
            throw bad("Visit not found.");
        }

        if (!visitPatientId.equals(patientId)) {
            throw bad("Visit does not belong to this patient.");
        }

        int sessionId = repository.findOrCreateSession(request.visitId());

        saveChoice(sessionId, Q_HANDLES_OBJECTS, request.handlesObjects());
        saveChoice(sessionId, Q_DRESSES, request.dresses());
        saveChoice(sessionId, Q_BATHES, request.bathes());
        saveChoice(sessionId, Q_TOILETS, request.toilets());
        saveMultiChoice(sessionId, Q_ARM_HAND_CONCERNS, request.armHandConcerns());
    }

    public record UeHistoryQuestionnaireResponse(
            String handlesObjects,
            String dresses,
            String bathes,
            String toilets,
            List<String> armHandConcerns) {}

    public UeHistoryQuestionnaireResponse find(int patientId, int visitId) {
        Integer visitPatientId = repository.findVisitPatientId(visitId);

        if (visitPatientId == null) {
            throw bad("Visit not found.");
        }

        if (!visitPatientId.equals(patientId)) {
            throw bad("Visit does not belong to this patient.");
        }

        Integer sessionId = repository.findExistingSessionId(visitId);

        if (sessionId == null) {
            return new UeHistoryQuestionnaireResponse(null, null, null, null, List.of());
        }

        return new UeHistoryQuestionnaireResponse(
            firstOrNull(repository.findActiveAnswerLabels(sessionId, Q_HANDLES_OBJECTS)),
            firstOrNull(repository.findActiveAnswerLabels(sessionId, Q_DRESSES)),
            firstOrNull(repository.findActiveAnswerLabels(sessionId, Q_BATHES)),
            firstOrNull(repository.findActiveAnswerLabels(sessionId, Q_TOILETS)),
            repository.findActiveAnswerLabels(sessionId, Q_ARM_HAND_CONCERNS));
    }

    private static String firstOrNull(List<String> values) {
        return values.isEmpty() ? null : values.get(0);
    }

    private void saveChoice(int sessionId, int questionId, String value) {
        if (value == null || value.isBlank()) {
            return;
        }

        Integer answerId = repository.resolveAnswerId(questionId, value);

        if (answerId == null) {
            throw bad("Could not resolve answer for question " + questionId + ": " + value);
        }

        repository.deactivateActiveAnswers(sessionId, questionId);
        repository.insertAnswer(sessionId, questionId, answerId, null);
    }

    private void saveMultiChoice(int sessionId, int questionId, List<String> values) {
        if (values == null || values.isEmpty()) {
            return;
        }

        repository.deactivateActiveAnswers(sessionId, questionId);

        for (String value : values) {
            Integer answerId = repository.resolveAnswerId(questionId, value);

            if (answerId == null) {
                throw bad("Could not resolve answer for question " + questionId + ": " + value);
            }

            repository.insertAnswer(sessionId, questionId, answerId, null);
        }
    }

    private static ResponseStatusException bad(String reason) {
        return new ResponseStatusException(HttpStatus.BAD_REQUEST, reason);
    }
}










package org.nemours.gaitlab.repository;

import java.util.List;
import java.util.Map;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Repository;

@Repository
public class FrequencyRepository {

    private final JdbcTemplate jdbc;

    public FrequencyRepository(JdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    // q-pt.jsp: SELECT * FROM frequencies ORDER BY code DESC
    public List<Map<String, Object>> findAllOrderByCodeDesc() {
        return jdbc.queryForList("SELECT code, freq, il8n_sp FROM public.frequencies ORDER BY code DESC");
    }
}












package org.nemours.gaitlab.repository;

import java.util.ArrayList;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Repository;

@Repository
public class HipQuestionnaireRepository {

    private final JdbcTemplate jdbc;

    public HipQuestionnaireRepository(JdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public Integer findVisitPatientId(int visitId) {
        List<Integer> ids = jdbc.queryForList(
            "SELECT pt_id FROM public.visits WHERE id = ?",
            Integer.class, visitId);

        return ids.isEmpty() ? null : ids.get(0);
    }

    public Map<String, Object> findByVisitId(int visitId) {
        List<Map<String, Object>> rows = jdbc.queryForList(
            "SELECT * FROM public.hip_score WHERE visit_id = ?", visitId);

        return rows.isEmpty() ? null : rows.get(0);
    }

    public void upsert(int visitId, LinkedHashMap<String, Object> columns) {
        List<String> names = new ArrayList<>(columns.keySet());

        String columnList = String.join(", ", names);
        String placeholders = names.stream().map(name -> "?").collect(Collectors.joining(", "));
        String updateAssignments = names.stream()
            .map(name -> name + " = EXCLUDED." + name)
            .collect(Collectors.joining(", "));

        List<Object> parameters = new ArrayList<>();
        parameters.add(visitId);
        parameters.addAll(columns.values());

        jdbc.update(
            "INSERT INTO public.hip_score (visit_id, " + columnList + ") "
                + "VALUES (?, " + placeholders + ") "
                + "ON CONFLICT (visit_id) DO UPDATE SET " + updateAssignments,
            parameters.toArray());
    }
}













package org.nemours.gaitlab.repository;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.stream.Collectors;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Repository;

@Repository
public class PodciQuestionnaireRepository {

    private static final Set<String> EXCLUDED_COLUMNS = Set.of(
        "id", "visit_id", "q2_type",
        "uepf_std", "uepf_norm",
        "tbm_std", "tbm_norm",
        "spf_std", "spf_norm",
        "pain_std", "pain_norm",
        "happiness_std", "happiness_norm",
        "global_std", "global_norm");

    private final JdbcTemplate jdbc;

    public PodciQuestionnaireRepository(JdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public Integer findVisitPatientId(int visitId) {
        List<Integer> ids = jdbc.queryForList(
            "SELECT pt_id FROM public.visits WHERE id = ?",
            Integer.class, visitId);

        return ids.isEmpty() ? null : ids.get(0);
    }

    public Map<String, String> findAnswerColumnTypes() {
        List<Map<String, Object>> rows = jdbc.queryForList(
            "SELECT column_name, data_type FROM information_schema.columns "
                + "WHERE table_schema = 'public' AND table_name = 'data_questionnaire'");

        Map<String, String> result = new HashMap<>();

        for (Map<String, Object> row : rows) {
            String name = (String) row.get("column_name");

            if (!EXCLUDED_COLUMNS.contains(name)) {
                result.put(name, (String) row.get("data_type"));
            }
        }

        return result;
    }

    private Integer findExistingRowId(int visitId, String variant) {
        List<Integer> ids = jdbc.queryForList(
            "SELECT id FROM public.data_questionnaire WHERE visit_id = ? AND q2_type = ?",
            Integer.class, visitId, variant);

        return ids.isEmpty() ? null : ids.get(0);
    }

    public Map<String, Object> findAnswers(int visitId, String variant) {
        List<Map<String, Object>> rows = jdbc.queryForList(
            "SELECT * FROM public.data_questionnaire WHERE visit_id = ? AND q2_type = ?",
            visitId, variant);

        if (rows.isEmpty()) {
            return Map.of();
        }

        Map<String, Object> row = rows.get(0);
        Map<String, Object> answers = new HashMap<>();

        for (Map.Entry<String, Object> entry : row.entrySet()) {
            if (!EXCLUDED_COLUMNS.contains(entry.getKey()) && entry.getValue() != null) {
                answers.put(entry.getKey(), entry.getValue());
            }
        }

        return answers;
    }

    public void upsert(int visitId, String variant, LinkedHashMap<String, Object> columns) {
        Integer existingId = findExistingRowId(visitId, variant);

        if (existingId == null) {
            List<String> names = new ArrayList<>();
            names.add("visit_id");
            names.add("q2_type");
            names.addAll(columns.keySet());

            List<Object> parameters = new ArrayList<>();
            parameters.add(visitId);
            parameters.add(variant);
            parameters.addAll(columns.values());

            String columnList = String.join(", ", names);
            String placeholders = names.stream().map(name -> "?").collect(Collectors.joining(", "));

            jdbc.update(
                "INSERT INTO public.data_questionnaire (" + columnList + ") VALUES (" + placeholders + ")",
                parameters.toArray());

        } else if (!columns.isEmpty()) {
            String assignments = columns.keySet().stream()
                .map(name -> name + " = ?")
                .collect(Collectors.joining(", "));

            List<Object> parameters = new ArrayList<>(columns.values());
            parameters.add(existingId);

            jdbc.update(
                "UPDATE public.data_questionnaire SET " + assignments + " WHERE id = ?",
                parameters.toArray());
        }
    }
}










package org.nemours.gaitlab.repository;

import java.sql.Date;
import java.util.ArrayList;
import java.util.Collections;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

import org.springframework.http.HttpStatus;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Repository;
import org.springframework.web.server.ResponseStatusException;

@Repository
public class QuestionnaireResponseRepository {

    private final JdbcTemplate jdbc;

    public QuestionnaireResponseRepository(JdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public void requirePatient(int patientId, boolean lock) {
        one("SELECT id " + "FROM public.patients " + "WHERE id = ?" + (lock ? " FOR UPDATE" : ""), patientId);
    }

    public Map<String, Object> findOwnedVisit( int patientId, int visitId, boolean lock) {
        return one("SELECT id, date::text AS date " + "FROM public.visits " + "WHERE id = ? " + "AND pt_id = ?" + (lock ? " FOR UPDATE" : ""), visitId, patientId);
    }

    public Map<String, Object> findPatientFields(int patientId, Map<String, String> mapping) {
        return one("SELECT " + selectColumns(mapping, "") + " FROM public.patients " + "WHERE id = ?", patientId);
    }

    public Map<String, Object> findVisitFields(int patientId, int visitId, Map<String, String> mapping) {
        return one("SELECT " + selectColumns(mapping, "") + " FROM public.visits " + "WHERE id = ? " + "AND pt_id = ?", visitId, patientId);
    }

    public List<Map<String, Object>> findListRows(int visitId, String table, Map<String, String> columns) {
        return jdbc.queryForList( "SELECT id, " + selectColumns(columns, "") + " FROM public." + table + " WHERE visit_id = ? " + "ORDER BY id", visitId);
    }

    public List<Map<String, Object>> findEligibleVisits(int patientId) {
        return jdbc.queryForList(
                """
                        SELECT
                            v.id,
                            v.date::text AS date,
                            COALESCE(
                                vt.name,
                                v.visit_type
                            ) AS "visitType",
                            tg.name AS "visitSubType"

                        FROM public.visits v

                        LEFT JOIN public.new_visit_type vt
                            ON vt.id = v.visit_type_id

                        LEFT JOIN public.new_test_group tg
                            ON tg.id = v.test_group_id

                        WHERE v.pt_id = ?

                          AND (
                              v.attendance IS NULL
                              OR v.attendance = ''
                              OR v.attendance = 'Yes'
                          )

                        ORDER BY
                            v.date ASC NULLS LAST,
                            v.id ASC
                        """,
                patientId);
    }

    public List<Map<String, Object>> findSameDayAnswers(int patientId, int visitId, Map<String, String> mapping) {
        return jdbc.queryForList(
                "SELECT " +
                        selectColumns(mapping, "v.") +
                        " " +
                        """
                                FROM public.visits selected

                                JOIN public.visits v
                                  ON v.pt_id = selected.pt_id
                                 AND (
                                     v.id = selected.id
                                     OR v.date = selected.date
                                 )

                                WHERE selected.id = ?
                                  AND selected.pt_id = ?

                                ORDER BY
                                    CASE
                                        WHEN v.id = selected.id
                                        THEN 0
                                        ELSE 1
                                    END,
                                    v.id
                                """,
                visitId, patientId);
    }

    public List<String> findRelationships() {
        return jdbc.queryForList("SELECT DISTINCT rship " + "FROM public.relationships " + "WHERE rship IS NOT NULL " + "ORDER BY rship", String.class);
    }

    public List<Map<String, Object>> findRelationships(String language) {
        return jdbc.queryForList("SELECT DISTINCT rship AS value, " + labelColumn(language, "rship")
            + " FROM public.relationships WHERE rship IS NOT NULL ORDER BY rship");
    }

    public List<String> findPrematureOptions() {
        return jdbc.queryForList("SELECT premtime " + "FROM public.premature " + "WHERE premtime IS NOT NULL " + "ORDER BY chron, premtime", String.class);
    }

    public List<Map<String, Object>> findPrematureOptions(String language) {
        return jdbc.queryForList("SELECT premtime AS value, " + labelColumn(language, "premtime")
            + " FROM public.premature WHERE premtime IS NOT NULL ORDER BY chron, premtime");
    }

    public List<String> findBirthStayOptions() {
        return jdbc.queryForList("SELECT birthstay " + "FROM public.birthstay " + "WHERE birthstay IS NOT NULL " + "ORDER BY chron, birthstay", String.class);
    }

    public List<Map<String, Object>> findBirthStayOptions(String language) {
        return jdbc.queryForList("SELECT birthstay AS value, " + labelColumn(language, "birthstay")
            + " FROM public.birthstay WHERE birthstay IS NOT NULL ORDER BY chron, birthstay");
    }

    public List<String> findAgeOptions() {
        return jdbc.queryForList("SELECT age " + "FROM public.ages1 " + "WHERE age IS NOT NULL " + "AND (chron < 2 OR chron > 9) " + "ORDER BY chron, age", String.class);
    }

    // public.ages1 has no il8n_sp column live (unlike the legacy dump's schema) — label is
    // always the English value until that's added.
    public List<Map<String, Object>> findAgeOptions(String language) {
        return jdbc.queryForList("SELECT age AS value, age AS label "
            + "FROM public.ages1 WHERE age IS NOT NULL AND (chron < 2 OR chron > 9) ORDER BY chron, age");
    }

    public List<Map<String, Object>> findLearningOptions() {
        return jdbc.queryForList("SELECT " + "code AS value, " + "description AS label " + "FROM public.learning " + "WHERE code IS NOT NULL " + "ORDER BY code");
    }

    public List<Map<String, Object>> findLearningOptions(String language) {
        return jdbc.queryForList("SELECT code AS value, " + labelColumn(language, "description")
            + " FROM public.learning WHERE code IS NOT NULL ORDER BY code");
    }

    public List<Map<String, Object>> findWalkingSupportOptions() {
        return jdbc.queryForList("SELECT " + "code AS value, " + "description AS label " + "FROM public.walk_support " + "ORDER BY code");
    }

    public List<Map<String, Object>> findWalkingSupportOptions(String language) {
        return jdbc.queryForList("SELECT code AS value, " + labelColumn(language, "description")
            + " FROM public.walk_support ORDER BY code");
    }

    public List<Map<String, Object>> findFmsOptions() {
        return jdbc.queryForList("SELECT " + "id AS value, " + "label, " + "description " + "FROM public.fms_list " + "WHERE label <> '' " + "ORDER BY sortorder, id");
    }

    // "description" stays the English long-form text in both languages; only the short "label" is translated.
    public List<Map<String, Object>> findFmsOptions(String language) {
        return jdbc.queryForList("SELECT id AS value, " + labelColumn(language, "label") + ", description "
            + "FROM public.fms_list WHERE label <> '' ORDER BY sortorder, id");
    }

    public List<String> findMedicationOptions() {
        return jdbc.queryForList("SELECT med " + "FROM public.seizure_meds " + "ORDER BY med", String.class);
    }

    // seizure_meds has no il8n_sp column in legacy — label is always the English value.
    public List<Map<String, Object>> findMedicationOptions(String language) {
        return jdbc.queryForList("SELECT med AS value, med AS label FROM public.seizure_meds ORDER BY med");
    }

    public List<String> findDeviceOptions(){
        return jdbc.queryForList("SELECT device " + "FROM public.devices " + "ORDER BY device", String.class);
    }

    public List<Map<String, Object>> findDeviceOptions(String language) {
        return jdbc.queryForList("SELECT device AS value, " + labelColumn(language, "device")
            + " FROM public.devices ORDER BY device");
    }

    public List<Map<String, Object>> findGaitConcernOptions(){
        return jdbc.queryForList("SELECT " + "code AS value, " + "description AS label " + "FROM public.gait_concerns " + "ORDER BY code");
    }

    public List<Map<String, Object>> findGaitConcernOptions(String language) {
        return jdbc.queryForList("SELECT code AS value, " + labelColumn(language, "description")
            + " FROM public.gait_concerns ORDER BY code");
    }

    public List<String> findPainOptions(){
        return jdbc.queryForList("SELECT part " + "FROM public.pain " + "ORDER BY part", String.class);
    }

    public List<Map<String, Object>> findPainOptions(String language) {
        return jdbc.queryForList("SELECT part AS value, " + labelColumn(language, "part")
            + " FROM public.pain ORDER BY part");
    }

    // "sp" reads the legacy il8n_sp column, falling back to English when it's blank/unset;
    // anything else (including plain "en") reads the original English column directly.
    private static String labelColumn(String language, String englishColumn) {
        return ("sp".equals(language)
            ? "COALESCE(NULLIF(il8n_sp, ''), " + englishColumn + ")"
            : englishColumn) + " AS label";
    }

    public boolean validLookup(String key, Object value) {
        String sql = switch (key) {
                case "historyRelationship", "relationship" -> "SELECT COUNT(*) " + "FROM public.relationships " + "WHERE rship = ?";
                case "premature" -> "SELECT COUNT(*) " + "FROM public.premature " + "WHERE premtime = ?";
                case "birthStay" -> "SELECT COUNT(*) " + "FROM public.birthstay " + "WHERE birthstay = ?";
                case "ageWalk" -> "SELECT COUNT(*) " + "FROM public.ages1 " + "WHERE age = ?";
                case "learning" -> "SELECT COUNT(*) " + "FROM public.learning " + "WHERE code = ?";
                case "walkingSupportCode" -> "SELECT COUNT(*) " + "FROM public.walk_support " + "WHERE code = ?";
                case "fms5Id", "fms50Id", "fms500Id" -> "SELECT COUNT(*) " + "FROM public.fms_list " + "WHERE id = ? " + "AND label <> ''";
                default -> null;
        };

        if (sql == null) {
                return true;
        }

        Long count = jdbc.queryForObject(sql, long.class, value);

        return count != null && count > 0;
    }

    public boolean gaitConcernExists(Object value) {
        Long count = jdbc.queryForObject("SELECT COUNT(*) " + "FROM public.gait_concerns " + "WHERE code = ?", Long.class, value);

        return count != null && count > 0;
    }

    public void updatePatientFields(int patientId, LinkedHashMap<String, String> mapping, LinkedHashMap<String, Object> values) {
        String assignments = mapping.values().stream().map(column -> column + " =?").collect(Collectors.joining(", "));
        List<Object> parameters = new ArrayList<>(values.values());
        int dateIndex = new ArrayList<>(mapping.keySet()).indexOf("baselineDate");
        if(dateIndex >=0){
                Object date = parameters.get(dateIndex);
                parameters.set(dateIndex, date == null || "".equals(date) ? null : Date.valueOf((String) date));
        }
        parameters.add(patientId);

        jdbc.update("UPDATE public.patients " + "SET " + assignments + " WHERE id = ?", parameters.toArray());
    }

    public void updateVisitFields(int patientId, int visitId, LinkedHashMap<String, String> mapping, LinkedHashMap<String, Object> values){
        String assignments = mapping.values().stream().map(column -> column + " =?").collect(Collectors.joining(", "));
        
        List<Object> parameters = new ArrayList<>(values.values());
        parameters.add(visitId);
        parameters.add(patientId);

        jdbc.update("UPDATE public.visits " + "SET " + assignments + " WHERE id = ? " + "AND pt_id = ?", parameters.toArray());
    }

    public void insertListRow(int visitId, String table, LinkedHashMap<String, String> columns, Map<String, Object> row) {
        List<Object> parameters = new ArrayList<>();
        for(String key : columns.keySet()) {
                parameters.add(row.get(key));
        }

        parameters.add(visitId);

        String placeholders = String.join(", ", Collections.nCopies(parameters.size(), "?"));

        jdbc.update("INSERT INTO public." + table + " (" + String.join(", ", columns.values()) + ", visit_id) VALUES (" + placeholders + ")", parameters.toArray());
    }

    public void updateListRow(int visitId, int rowId, String table, LinkedHashMap<String, String> columns, Map<String, Object> row) {
        List<Object> parameters = new ArrayList<>();
        for(String key : columns.keySet()) {
                parameters.add(row.get(key));
        }

        parameters.add(visitId);
        parameters.add(rowId);

        String setters = columns.values().stream().map(column -> column + " =?").collect(Collectors.joining(", "));
        jdbc.update("UPDATE public." + table + " SET " + setters + " WHERE visit_id = ? " + "AND id = ?", parameters.toArray());
    }

    public void deleteListRow(int visitId, int rowId, String table) {
        jdbc.update("DELETE FROM public." + table + " WHERE visit_id = ? " + "AND id = ?", visitId, rowId);
    }

    private Map<String, Object> one(String sql, Object... parameters) {
        List<Map<String, Object>> rows = jdbc.queryForList(sql, parameters);

        if(rows.isEmpty()) {
                throw new ResponseStatusException(HttpStatus.NOT_FOUND, "Patient or visit was not found.");
        }

        Map<String, Object> result = new LinkedHashMap<>(rows.get(0));

        result.replaceAll((key, value) -> value instanceof Date date ? date.toLocalDate().toString() : value);

        return result;
    }

    private String selectColumns(Map<String, String> columns, String prefix) {
        return columns.entrySet().stream().map(entry -> prefix + entry.getValue() + " AS \"" + entry.getKey() + "\"").collect(Collectors.joining(", "));
    }
}











package org.nemours.gaitlab.repository;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Repository;

@Repository
public class QuestionnaireTrackingRepository {

    private final JdbcTemplate jdbc;

    public QuestionnaireTrackingRepository(JdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    // Skips the insert if a row for (visit_id, descriptor) already exists, so re-tracking the
    // same assembled list after a re-save doesn't duplicate q_track rows.
    public void insertIfAbsent(int visitId, String descriptor, Integer sessionId) {
        jdbc.update(
            "INSERT INTO public.q_track (visit_id, descriptor, time, session_id) "
                + "SELECT ?, ?, now(), ? "
                + "WHERE NOT EXISTS ("
                + "  SELECT 1 FROM public.q_track WHERE visit_id = ? AND descriptor = ?"
                + ")",
            visitId, descriptor, sessionId, visitId, descriptor);
    }
}











package org.nemours.gaitlab.repository;

import java.util.List;
import java.util.Map;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Repository;

@Repository
public class SportsQuestionnaireRepository {

    private static final int QUESTIONNAIRE_ID = 1;
    private static final int LANGUAGE_ID = 1;

    private final JdbcTemplate jdbc;

    public SportsQuestionnaireRepository(JdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public Integer findVisitPatientId(int visitId) {
        List<Integer> ids = jdbc.queryForList(
            "SELECT pt_id FROM public.visits WHERE id = ?",
            Integer.class, visitId);

        return ids.isEmpty() ? null : ids.get(0);
    }

    public int findOrCreateSession(int visitId) {
        List<Integer> ids = jdbc.queryForList(
            "SELECT id FROM nmquestionnaire.session WHERE visit_id = ? AND questionnaire_id = ?",
            Integer.class, visitId, QUESTIONNAIRE_ID);

        if (!ids.isEmpty()) {
            return ids.get(0);
        }

        // Legacy never calls SessionServiceImpl.complete() for Sports (questionnaire_id=1) — that's
        // only reachable from QuestionTakeAction for Struts questionnaires 2-5. Leave complete null.
        return jdbc.queryForObject(
            "INSERT INTO nmquestionnaire.session (visit_id, questionnaire_id, language_id, begin) "
                + "VALUES (?, ?, ?, now()) RETURNING id",
            Integer.class, visitId, QUESTIONNAIRE_ID, LANGUAGE_ID);
    }

    public Integer findExistingSessionId(int visitId) {
        List<Integer> ids = jdbc.queryForList(
            "SELECT id FROM nmquestionnaire.session WHERE visit_id = ? AND questionnaire_id = ?",
            Integer.class, visitId, QUESTIONNAIRE_ID);

        return ids.isEmpty() ? null : ids.get(0);
    }

    // The single "free text" placeholder answer legacy attaches free-text patient_answer rows to,
    // tagged TextAnswerContainer (matches legacy's answer ids 10/37/462 for questions 2/8/60,
    // reseeded here with fresh auto-generated ids since live ids don't mirror the dump's numbering).
    public Integer findTextAnswerId(int questionId) {
        List<Integer> ids = jdbc.queryForList(
            "SELECT id FROM nmquestionnaire.answer WHERE question_id = ? AND tag = 'TextAnswerContainer'",
            Integer.class, questionId);

        return ids.isEmpty() ? null : ids.get(0);
    }

    public Integer resolveAnswerId(int questionId, String text) {
        List<Integer> ids = jdbc.queryForList(
            "SELECT a.id FROM nmquestionnaire.answer a "
                + "JOIN nmquestionnaire.answer_text at ON at.answer_id = a.id "
                + "WHERE a.question_id = ? AND at.language_id = ? AND at.text = ?",
            Integer.class, questionId, LANGUAGE_ID, text);

        return ids.isEmpty() ? null : ids.get(0);
    }

    public void deactivateActiveAnswers(int sessionId, int questionId) {
        jdbc.update(
            "UPDATE nmquestionnaire.patient_answer SET active = false "
                + "WHERE session_id = ? AND question_id = ? AND active = true",
            sessionId, questionId);
    }

    public void insertAnswer(int sessionId, int questionId, Integer answerId, String answerText) {
        jdbc.update(
            "INSERT INTO nmquestionnaire.patient_answer "
                + "(session_id, question_id, answer_id, answer_text, time, active) "
                + "VALUES (?, ?, ?, ?, now(), true)",
            sessionId, questionId, answerId, answerText);
    }

    // Active answer labels (English) for a single/multi-choice question, in insertion order.
    public List<String> findActiveAnswerLabels(int sessionId, int questionId) {
        return jdbc.queryForList(
            "SELECT at.text FROM nmquestionnaire.patient_answer pa "
                + "JOIN nmquestionnaire.answer_text at ON at.answer_id = pa.answer_id AND at.language_id = ? "
                + "WHERE pa.session_id = ? AND pa.question_id = ? AND pa.active = true "
                + "ORDER BY pa.id",
            String.class, LANGUAGE_ID, sessionId, questionId);
    }

    // The raw text saved against a free-text question's TextAnswerContainer placeholder answer.
    public String findActiveFreeText(int sessionId, int questionId) {
        List<String> texts = jdbc.queryForList(
            "SELECT answer_text FROM nmquestionnaire.patient_answer "
                + "WHERE session_id = ? AND question_id = ? AND active = true "
                + "ORDER BY id DESC LIMIT 1",
            String.class, sessionId, questionId);

        return texts.isEmpty() ? null : texts.get(0);
    }

    // Mile time (Q7) stores both the "Unknown"/"Time" choice (via answer_id) and, when known,
    // the raw mile time (via answer_text on that same row) — needs both columns in one read.
    public Map<String, Object> findActiveChoiceWithText(int sessionId, int questionId) {
        List<Map<String, Object>> rows = jdbc.queryForList(
            "SELECT at.text AS label, pa.answer_text AS free_text "
                + "FROM nmquestionnaire.patient_answer pa "
                + "JOIN nmquestionnaire.answer_text at ON at.answer_id = pa.answer_id AND at.language_id = ? "
                + "WHERE pa.session_id = ? AND pa.question_id = ? AND pa.active = true "
                + "ORDER BY pa.id DESC LIMIT 1",
            LANGUAGE_ID, sessionId, questionId);

        return rows.isEmpty() ? null : rows.get(0);
    }
}











package org.nemours.gaitlab.repository;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Repository;

@Repository
public class UeHistoryQuestionnaireRepository {

    private static final int QUESTIONNAIRE_ID = 7;
    private static final int LANGUAGE_ID = 1;

    private final JdbcTemplate jdbc;

    public UeHistoryQuestionnaireRepository(JdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public Integer findVisitPatientId(int visitId) {
        List<Integer> ids = jdbc.queryForList(
            "SELECT pt_id FROM public.visits WHERE id = ?",
            Integer.class, visitId);

        return ids.isEmpty() ? null : ids.get(0);
    }

    public int findOrCreateSession(int visitId) {
        List<Integer> ids = jdbc.queryForList(
            "SELECT id FROM nmquestionnaire.session WHERE visit_id = ? AND questionnaire_id = ?",
            Integer.class, visitId, QUESTIONNAIRE_ID);

        if (!ids.isEmpty()) {
            return ids.get(0);
        }

        // Matches Sports: legacy's SessionServiceImpl.complete() is never reachable for this
        // questionnaire either (QuestionTakeAction only wires it up for Struts ids 2-5).
        return jdbc.queryForObject(
            "INSERT INTO nmquestionnaire.session (visit_id, questionnaire_id, language_id, begin) "
                + "VALUES (?, ?, ?, now()) RETURNING id",
            Integer.class, visitId, QUESTIONNAIRE_ID, LANGUAGE_ID);
    }

    public Integer findExistingSessionId(int visitId) {
        List<Integer> ids = jdbc.queryForList(
            "SELECT id FROM nmquestionnaire.session WHERE visit_id = ? AND questionnaire_id = ?",
            Integer.class, visitId, QUESTIONNAIRE_ID);

        return ids.isEmpty() ? null : ids.get(0);
    }

    public Integer resolveAnswerId(int questionId, String text) {
        List<Integer> ids = jdbc.queryForList(
            "SELECT a.id FROM nmquestionnaire.answer a "
                + "JOIN nmquestionnaire.answer_text at ON at.answer_id = a.id "
                + "WHERE a.question_id = ? AND at.language_id = ? AND at.text = ?",
            Integer.class, questionId, LANGUAGE_ID, text);

        return ids.isEmpty() ? null : ids.get(0);
    }

    public void deactivateActiveAnswers(int sessionId, int questionId) {
        jdbc.update(
            "UPDATE nmquestionnaire.patient_answer SET active = false "
                + "WHERE session_id = ? AND question_id = ? AND active = true",
            sessionId, questionId);
    }

    public void insertAnswer(int sessionId, int questionId, Integer answerId, String answerText) {
        jdbc.update(
            "INSERT INTO nmquestionnaire.patient_answer "
                + "(session_id, question_id, answer_id, answer_text, time, active) "
                + "VALUES (?, ?, ?, ?, now(), true)",
            sessionId, questionId, answerId, answerText);
    }

    // Active answer labels (English) for a question, in insertion order. Empty if none saved yet.
    public List<String> findActiveAnswerLabels(int sessionId, int questionId) {
        return jdbc.queryForList(
            "SELECT at.text FROM nmquestionnaire.patient_answer pa "
                + "JOIN nmquestionnaire.answer_text at ON at.answer_id = pa.answer_id AND at.language_id = ? "
                + "WHERE pa.session_id = ? AND pa.question_id = ? AND pa.active = true "
                + "ORDER BY pa.id",
            String.class, LANGUAGE_ID, sessionId, questionId);
    }

    // Bilingual question text + ordered answer options, straight from nmquestionnaire, for the
    // /api/reference/ue-history-questions endpoint. "value" is always the English answer text,
    // since that's what resolveAnswerId() matches on and what save() expects back.
    public Map<String, Object> findQuestionBundle(int questionId) {
        List<Map<String, Object>> textRows = jdbc.queryForList(
            "SELECT language_id, text FROM nmquestionnaire.question_text WHERE question_id = ?",
            questionId);

        String questionEn = null;
        String questionSp = null;

        for (Map<String, Object> row : textRows) {
            int languageId = ((Number) row.get("language_id")).intValue();
            String text = (String) row.get("text");

            if (languageId == 1) {
                questionEn = text;
            } else if (languageId == 2) {
                questionSp = text;
            }
        }

        List<Map<String, Object>> answerRows = jdbc.queryForList(
            "SELECT a.id, at.language_id, at.text FROM nmquestionnaire.answer a "
                + "LEFT JOIN nmquestionnaire.answer_text at ON at.answer_id = a.id "
                + "WHERE a.question_id = ? "
                + "ORDER BY a.sortorder, a.id",
            questionId);

        LinkedHashMap<Integer, Map<String, Object>> optionsByAnswerId = new LinkedHashMap<>();

        for (Map<String, Object> row : answerRows) {
            int answerId = ((Number) row.get("id")).intValue();
            Object languageIdValue = row.get("language_id");
            String text = (String) row.get("text");

            Map<String, Object> option = optionsByAnswerId.computeIfAbsent(answerId, id -> new LinkedHashMap<>());

            if (languageIdValue == null) {
                continue;
            }

            int languageId = ((Number) languageIdValue).intValue();

            if (languageId == 1) {
                option.put("value", text);
                option.put("en", text);
            } else if (languageId == 2) {
                option.put("sp", text);
            }
        }

        Map<String, Object> question = new LinkedHashMap<>();
        question.put("en", questionEn);
        question.put("sp", questionSp);

        Map<String, Object> bundle = new LinkedHashMap<>();
        bundle.put("question", question);
        bundle.put("options", optionsByAnswerId.values().stream()
            .filter(option -> option.get("value") != null)
            .toList());

        return bundle;
    }
}










package org.nemours.gaitlab.requests;

import java.util.List;

public record AssembleQuestionnaireRequest(
    Integer visitId,
    List<String> selections,
    String language
) {}








package org.nemours.gaitlab.requests;

import java.util.Map;

public record HipQuestionnaireRequest(
    Integer visitId,
    Map<String, Integer> womac,
    Integer ucla,
    Map<String, Integer> harris
) {}








package org.nemours.gaitlab.requests;

import java.util.Map;

public record PodciQuestionnaireRequest(
    Integer visitId,
    String variant,
    Map<String, Object> answers
) {}








package org.nemours.gaitlab.requests;

import java.util.List;

public record SportsQuestionnaireRequest(
    Integer visitId,
    String reportingPerson,
    String physicalTherapy,
    String competitiveSports,
    String yearsInSport,
    String daysPerWeek,
    String orthotics,
    List<String> painLocations,
    String mileTimeKnown,
    String mileTime,
    String goals
) {}








package org.nemours.gaitlab.requests;

import java.util.List;

public record UeHistoryQuestionnaireRequest(
    Integer visitId,
    String handlesObjects,
    String dresses,
    String bathes,
    String toilets,
    List<String> armHandConcerns
) {}








package org.nemours.gaitlab.requests;

import java.util.List;
import java.util.Map;

public record UpdateQuestionnaireResponseRequest(
        String version,
        Map<String, Object> fields,
        Map<String, List<Map<String, Object>>> lists
) {
}





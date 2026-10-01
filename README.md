session

<div class="session-page">

  @if (assembling) {

    <main class="simple-state">

      <div class="status-card">

        <h2>
          Preparing Questionnaires
        </h2>

        <p>
          Please wait...
        </p>

      </div>

    </main>

  } @else if (assembleError) {

    <main class="simple-state">

      <div class="status-card">

        <h2>
          Something went wrong
        </h2>

        <div class="state-error">
          {{ assembleError }}
        </div>

        <button
          type="button"
          class="primary-button"
          (click)="retry()"
        >
          Retry
        </button>

      </div>

    </main>

  } @else if (showThankYou) {

    <app-questionnaire-thank-you
      (closeWindow)="closeWindow()"
    >
    </app-questionnaire-thank-you>

  } @else if (showWelcome) {

    <app-questionnaire-welcome
      [patientName]="patientDisplayName"
      [questionnaireNames]="questionnaireNames"
      (startQuestionnaire)="startQuestionnaires()"
    >
    </app-questionnaire-welcome>

  } @else if (currentStep) {

    <main class="questionnaire-shell">

      <section class="questionnaire-card">

        <header class="questionnaire-header">

          <div class="progress-label">
            {{ progressText }}
          </div>

          <h1>
            {{ currentStep.title }}
          </h1>

        </header>


        <div
          class="question-scroll"
          (input)="scheduleProgressRefresh()"
          (change)="scheduleProgressRefresh()"
          (click)="scheduleProgressRefresh()"
        >

          @if (
            currentStep.key ===
            'HISTORY'
          ) {

            @if (showFirstVisit) {

              <app-first-visit-questionnaire
                #sectionCmp
                [patientId]="patientId"
                [firstName]="firstName"
                [language]="languageCode"
                [hideActions]="true"
                (baselineChange)="onBaselineChange($event)"
                (progressChange)="scheduleProgressRefresh()"
                (saveComplete)="onSectionSaved()"
                (saveFailed)="onSectionSaveFailed($event)"
              >
              </app-first-visit-questionnaire>

            }


            <app-history-questionnaire
              #sectionCmp
              [patientId]="patientId"
              [visitId]="historyVisitId"
              [firstName]="firstName"
              [language]="languageCode"
              [showWho]="showWho"
              [showGeneral]="showGeneral"
              [showGait]="showGait"
              [showConcerns]="showConcerns"
              [copyWho]="baseline"
              [hideActions]="true"
              (progressChange)="scheduleProgressRefresh()"
              (saveComplete)="onSectionSaved()"
              (saveFailed)="onSectionSaveFailed($event)"
            >
            </app-history-questionnaire>

          } @else if (
            currentStep.key ===
            'HIP'
          ) {

            <app-hip-questionnaire
              #sectionCmp
              [patientId]="patientId"
              [visitId]="currentStep.visitId"
              [hideActions]="true"
              (progressChange)="scheduleProgressRefresh()"
              (saveComplete)="onSectionSaved()"
              (saveFailed)="onSectionSaveFailed($event)"
            >
            </app-hip-questionnaire>

          } @else if (
            currentStep.key ===
            'SPORTS'
          ) {

            <app-sports-questionnaire
              #sectionCmp
              [patientId]="patientId"
              [visitId]="currentStep.visitId"
              [hideActions]="true"
              (progressChange)="scheduleProgressRefresh()"
              (saveComplete)="onSectionSaved()"
              (saveFailed)="onSectionSaveFailed($event)"
            >
            </app-sports-questionnaire>

          } @else if (
            currentStep.key === 'PODCI_CH' ||
            currentStep.key === 'PODCI_AP' ||
            currentStep.key === 'PODCI_AS'
          ) {

            <app-podci-questionnaire
              #sectionCmp
              [patientId]="patientId"
              [visitId]="currentStep.visitId"
              [firstName]="firstName"
              [language]="languageCode"
              [variant]="podciVariant(currentStep.key)"
              [hideActions]="true"
              (progressChange)="scheduleProgressRefresh()"
              (saveComplete)="onSectionSaved()"
              (saveFailed)="onSectionSaveFailed($event)"
            >
            </app-podci-questionnaire>

          } @else if (
            currentStep.key ===
            'UE_HISTORY'
          ) {

            <app-ue-history-questionnaire
              #sectionCmp
              [patientId]="patientId"
              [visitId]="currentStep.visitId"
              [hideActions]="true"
              (progressChange)="scheduleProgressRefresh()"
              (saveComplete)="onSectionSaved()"
              (saveFailed)="onSectionSaveFailed($event)"
            >
            </app-ue-history-questionnaire>

          }

        </div>

      </section>


      <aside class="information-card">

        <app-questionnaire-patient-info
          [name]="patientDisplayName"
          [dob]="patientDob"
          [sex]="patientSex"
          [mrn]="patientMrn"
        >
        </app-questionnaire-patient-info>


        <app-questionnaire-progress
          [questionnaires]="progressItems"
          [currentKey]="currentStep.key"
        >
        </app-questionnaire-progress>

      </aside>


      <footer class="session-footer">

        <div class="footer-left">

          <button
            type="button"
            class="secondary-button"
            [disabled]="saving"
            (click)="requestSaveAndClose()"
          >
            Save and Close
          </button>

          @if (saveError) {

            <div class="footer-error">
              {{ saveError }}
            </div>

          }

        </div>


        <div class="footer-right">

          @if (!isLastQuestionnaire) {

            <button
              type="button"
              class="primary-button"
              [disabled]="saving"
              (click)="nextQuestionnaire()"
            >
              {{
                saving
                  ? 'Saving...'
                  : 'Next'
              }}
            </button>

          } @else {

            <button
              type="button"
              class="primary-button"
              [disabled]="saving"
              (click)="submitQuestionnaires()"
            >
              {{
                saving
                  ? 'Submitting...'
                  : 'Submit'
              }}
            </button>

          }

        </div>

      </footer>

    </main>

  }


  @if (showClosePopup) {

    <app-questionnaire-close
      [remainingCount]="remainingCount"
      [saving]="saving"
      (continueQuestionnaire)="continueQuestionnaire()"
      (saveAndClose)="confirmSaveAndClose()"
    >
    </app-questionnaire-close>

  }

</div>








:host {
  display: block;
  height: 100vh;
}

.session-page {
  height: 100vh;
  overflow: hidden;
  background: #f7f6f3;
  color: #101a48;
}

.simple-state {
  width: min(760px, calc(100% - 32px));
  margin: 0 auto;
  padding-top: 48px;
}

.status-card {
  padding: 30px;
  border: 1px solid #dfe5e8;
  border-radius: 10px;
  background: #ffffff;
  box-sizing: border-box;
}

.status-card h2 {
  margin: 0 0 10px;
  font-size: 22px;
}

.status-card p {
  margin: 0;
  color: #61707a;
}

.state-error {
  margin: 16px 0;
  padding: 11px 14px;
  border: 1px solid #e3a9a9;
  border-radius: 6px;
  background: #fff3f3;
  color: #a12626;
  font-size: 13px;
}

.questionnaire-shell {
  width: min(940px, calc(100vw - 32px));
  height: calc(100vh - 28px);
  margin: 14px auto;

  display: grid;

  grid-template-columns:
    minmax(0, 2fr)
    minmax(280px, 0.95fr);

  grid-template-rows:
    minmax(0, 1fr)
    58px;

  gap: 10px;
}

.questionnaire-card,
.information-card {
  min-height: 0;
  border: 1px solid #dde2e6;
  border-radius: 9px;
  background: #ffffff;
  box-sizing: border-box;
}

.questionnaire-card {
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.questionnaire-header {
  flex: 0 0 auto;
  padding: 20px 24px 12px;
  background: #ffffff;
}

.progress-label {
  margin-bottom: 5px;
  color: #101a48;
  font-size: 12px;
  font-weight: 600;
  text-transform: uppercase;
}

.questionnaire-header h1 {
  margin: 0;
  color: #101a48;
  font-size: 28px;
  font-weight: 700;
  line-height: 1.15;
}

.question-scroll {
  min-height: 0;
  flex: 1 1 auto;
  overflow-y: auto;
  padding: 0 20px 24px;
  box-sizing: border-box;
  scrollbar-gutter: stable;
}

.question-scroll::-webkit-scrollbar,
.information-card::-webkit-scrollbar {
  width: 8px;
}

.question-scroll::-webkit-scrollbar-track,
.information-card::-webkit-scrollbar-track {
  background: transparent;
}

.question-scroll::-webkit-scrollbar-thumb,
.information-card::-webkit-scrollbar-thumb {
  border-radius: 10px;
  background: #c5cbd0;
}

.information-card {
  overflow-y: auto;
  padding: 28px 24px;
}

.session-footer {
  grid-column: 1 / -1;

  display: flex;
  align-items: center;
  justify-content: space-between;

  gap: 18px;
  min-width: 0;
}

.footer-left,
.footer-right {
  display: flex;
  align-items: center;
  gap: 12px;
}

.footer-left {
  min-width: 0;
  flex: 1;
}

.footer-right {
  flex: 0 0 auto;
}

.footer-error {
  min-width: 0;
  padding: 9px 12px;
  border: 1px solid #e3a9a9;
  border-radius: 6px;
  background: #fff3f3;
  color: #a12626;
  font-size: 12px;
  line-height: 1.35;
}

.primary-button,
.secondary-button {
  height: 44px;
  min-width: 150px;
  padding: 0 24px;
  border-radius: 6px;
  font-family: inherit;
  font-size: 14px;
  font-weight: 600;
  white-space: nowrap;
  cursor: pointer;
}

.primary-button {
  border: 1px solid #008779;
  background: #009688;
  color: #ffffff;
}

.primary-button:hover:not(:disabled) {
  background: #00796b;
}

.secondary-button {
  border: 1px solid #aeb8bf;
  background: #ffffff;
  color: #17214f;
}

.secondary-button:hover:not(:disabled) {
  background: #f3f6f7;
}

.primary-button:disabled,
.secondary-button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

@media (max-width: 820px) {

  .questionnaire-shell {
    width: min(
      760px,
      calc(100vw - 20px)
    );

    grid-template-columns:
      minmax(0, 1.75fr)
      minmax(240px, 1fr);
  }

  .questionnaire-header {
    padding: 18px 18px 10px;
  }

  .questionnaire-header h1 {
    font-size: 24px;
  }

  .question-scroll {
    padding: 0 14px 20px;
  }

  .information-card {
    padding: 22px 18px;
  }
}





questionnaire-patient-info.ts


import { CommonModule } from '@angular/common';
import {
  Component,
  Input
} from '@angular/core';

@Component({
  selector: 'app-questionnaire-patient-info',
  standalone: true,
  imports: [
    CommonModule
  ],
  templateUrl:
    './questionnaire-patient-info.html',
  styleUrl:
    './questionnaire-patient-info.css'
})
export class QuestionnairePatientInfo {

  @Input()
  name = '';

  @Input()
  dob = '';

  @Input()
  sex = '';

  @Input()
  mrn = '';

  get displaySex():
    string {

    const value =
      (this.sex ?? '')
        .trim()
        .toUpperCase();

    if (
      value === 'M' ||
      value === 'MALE'
    ) {
      return 'Male';
    }

    if (
      value === 'F' ||
      value === 'FEMALE'
    ) {
      return 'Female';
    }

    return (
      this.sex ||
      '-'
    );
  }
}









<section class="patient-info">

  <h2>
    Patient Information
  </h2>

  <div class="info-row">

    <span>
      Name
    </span>

    <strong>
      {{ name || '-' }}
    </strong>

  </div>


  <div class="info-row">

    <span>
      Date of Birth
    </span>

    <strong>
      {{
        dob
          ? (dob | date:'MM/dd/yyyy')
          : '-'
      }}
    </strong>

  </div>


  <div class="info-row">

    <span>
      Sex
    </span>

    <strong>
      {{ displaySex }}
    </strong>

  </div>


  <div class="info-row">

    <span>
      MRN
    </span>

    <strong>
      {{ mrn || '-' }}
    </strong>

  </div>

</section>









.patient-info {
  padding-bottom: 24px;
  border-bottom: 1px solid #dfe5e8;
}

.patient-info h2 {
  margin: 0 0 24px;
  color: #101a48;
  font-size: 22px;
  font-weight: 700;
}

.info-row {
  display: grid;

  grid-template-columns:
    minmax(0, 1fr)
    minmax(0, 1.1fr);

  gap: 14px;

  margin-bottom: 18px;

  color: #17214f;

  font-size: 15px;
  line-height: 1.35;
}

.info-row:last-child {
  margin-bottom: 0;
}

.info-row strong {
  overflow-wrap: anywhere;
  font-weight: 700;
}








questionnaire-progress.ts


import { CommonModule } from '@angular/common';

import {
  Component,
  Input
} from '@angular/core';

import {
  QuestionnaireProgressItem
} from '../questionnaire-progress.model';

@Component({
  selector: 'app-questionnaire-progress',
  standalone: true,
  imports: [
    CommonModule
  ],
  templateUrl:
    './questionnaire-progress.html',
  styleUrl:
    './questionnaire-progress.css'
})
export class QuestionnaireProgress {

  @Input()
  questionnaires:
    QuestionnaireProgressItem[] = [];

  @Input()
  currentKey = '';

  get current():
    QuestionnaireProgressItem | null {

    return (
      this.questionnaires.find(
        item =>
          item.key ===
          this.currentKey
      ) ?? null
    );
  }

  get percentage():
    number {

    if (
      !this.current ||
      this.current.total <= 0
    ) {
      return 0;
    }

    return Math.min(
      100,
      Math.round(
        (
          this.current.answered /
          this.current.total
        ) * 100
      )
    );
  }

  get circleBackground():
    string {

    return (
      `conic-gradient(` +
      `#009688 0 ${this.percentage}%, ` +
      `#e5e9ed ${this.percentage}% 100%)`
    );
  }
}









<section class="progress-panel">

  @if (current) {

    <div
      class="progress-circle"
      [style.background]="circleBackground"
    >

      <div class="progress-center">

        <div class="answered-count">
          {{ current.answered }}
        </div>

        <div class="total-count">
          of {{ current.total }}
        </div>

        <div class="answered-label">
          Answered
        </div>

      </div>

    </div>

  }


  <div class="divider"></div>


  <h2>
    Questionnaires
  </h2>


  <div class="questionnaire-list">

    @for (
      questionnaire of questionnaires;
      track questionnaire.key
    ) {

      <div
        class="questionnaire-row"
        [class.current]="
          questionnaire.key ===
          currentKey
        "
        [class.completed]="
          questionnaire.completed
        "
      >

        <strong>
          {{ questionnaire.name }}
        </strong>

        <span>

          @if (
            questionnaire.completed
          ) {

            Completed

          } @else {

            {{ questionnaire.answered }}
            of
            {{ questionnaire.total }}

          }

        </span>

      </div>

    }

  </div>

</section>








.progress-panel {
  padding-top: 28px;
}

.progress-circle {
  position: relative;

  width: 205px;
  height: 205px;

  margin: 0 auto;

  border-radius: 50%;
}

.progress-circle::before {
  content: '';

  position: absolute;

  inset: 25px;

  border-radius: 50%;

  background: #ffffff;
}

.progress-center {
  position: absolute;

  inset: 0;

  z-index: 1;

  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;

  color: #101a48;
}

.answered-count {
  font-size: 40px;
  font-weight: 700;
  line-height: 1;
}

.total-count {
  margin-top: 8px;

  font-size: 16px;
  font-weight: 600;
}

.answered-label {
  margin-top: 4px;

  font-size: 14px;
  font-weight: 600;
}

.divider {
  height: 1px;

  margin: 32px 0;

  background: #dfe5e8;
}

.progress-panel h2 {
  margin: 0 0 18px;

  color: #101a48;

  font-size: 22px;
  font-weight: 700;
}

.questionnaire-list {
  display: flex;
  flex-direction: column;

  gap: 10px;
}

.questionnaire-row {
  display: flex;
  align-items: center;
  justify-content: space-between;

  gap: 12px;

  min-height: 58px;

  padding: 0 16px;

  border: 1px solid #dfe5e8;
  border-radius: 7px;

  background: #ffffff;

  color: #15204d;

  box-sizing: border-box;
}

.questionnaire-row.current {
  border-color: #9bd2ca;
  background: #f3fbf9;
}

.questionnaire-row.completed {
  border-color: #9bd2ca;
}

.questionnaire-row strong {
  min-width: 0;

  font-size: 15px;
}

.questionnaire-row span {
  flex-shrink: 0;

  font-size: 13px;

  text-align: right;
}

@media (max-width: 900px) {

  .progress-circle {
    width: 170px;
    height: 170px;
  }

  .progress-circle::before {
    inset: 21px;
  }

  .answered-count {
    font-size: 34px;
  }
}








questionnaire-welcome.ts


import {
  Component,
  EventEmitter,
  Input,
  Output
} from '@angular/core';

import {
  CommonModule
} from '@angular/common';

@Component({
  selector: 'app-questionnaire-welcome',
  standalone: true,
  imports: [
    CommonModule
  ],
  templateUrl:
    './questionnaire-welcome.html',
  styleUrl:
    './questionnaire-welcome.css'
})
export class QuestionnaireWelcome {

  @Input()
  patientName = '';

  @Input()
  questionnaireNames:
    string[] = [];

  @Output()
  startQuestionnaire =
    new EventEmitter<void>();

  start(): void {
    this.startQuestionnaire.emit();
  }
}










<div class="welcome-page">

  <div class="welcome-card">

    <img
      src="/questionnaire.jpeg"
      alt="Gait and Motion Analysis Laboratory Patient Questionnaire"
      class="welcome-banner"
    />

    <div class="welcome-content">

      <p class="intro-text">
        In order to better understand our patients,
        we need your help with some history about
        {{ patientName }}.
      </p>

      <div class="questionnaire-section">

        <p class="section-title">
          The following questionnaires need to be completed:
        </p>

        <ul class="questionnaire-list">

          @for (
            questionnaire of questionnaireNames;
            track questionnaire
          ) {

            <li>
              {{ questionnaire }}
            </li>

          }

        </ul>

      </div>

      <p class="info-text">
        If you have any questions while completing the questionnaire,
        please don't hesitate to ask a member of our staff for help.
      </p>

      <p class="info-text">
        These questions will be asked on each visit so we can stay up
        to date with you.
      </p>

      <div class="welcome-actions">

        <button
          type="button"
          class="start-button"
          (click)="start()"
        >
          Start
        </button>

      </div>

    </div>

  </div>

</div>









.welcome-page {
  width: 100%;
  display: flex;
  justify-content: center;
  padding: 28px 24px 50px;
  box-sizing: border-box;
}

.welcome-card {
  width: 100%;
  max-width: 1000px;
  background: #ffffff;
  border: 1px solid #dfe5e8;
  border-radius: 10px;
  overflow: hidden;
  box-sizing: border-box;
}

.welcome-banner {
  display: block;
  width: 100%;
  height: auto;
  object-fit: cover;
}

.welcome-content {
  padding: 34px 40px 36px;
}

.intro-text {
  margin: 0 0 28px;
  color: #37474f;
  font-size: 16px;
  line-height: 1.65;
}

.questionnaire-section {
  margin-bottom: 28px;
}

.section-title {
  margin: 0 0 12px;
  color: #263238;
  font-size: 16px;
  font-weight: 600;
}

.questionnaire-list {
  margin: 0;
  padding-left: 24px;
}

.questionnaire-list li {
  margin-bottom: 8px;
  color: #37474f;
  font-size: 15px;
  line-height: 1.5;
}

.info-text {
  margin: 0 0 20px;
  color: #546168;
  font-size: 15px;
  line-height: 1.65;
}

.welcome-actions {
  display: flex;
  justify-content: flex-end;
  margin-top: 32px;
}

.start-button {
  min-width: 120px;
  height: 42px;
  padding: 0 26px;
  border: none;
  border-radius: 6px;
  background: #009688;
  color: #ffffff;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
}

.start-button:hover {
  background: #00796b;
}

@media (max-width: 700px) {

  .welcome-page {
    padding: 18px 14px 40px;
  }

  .welcome-content {
    padding: 24px 20px 28px;
  }

  .intro-text,
  .section-title,
  .info-text {
    font-size: 14px;
  }

  .questionnaire-list li {
    font-size: 14px;
  }

  .welcome-actions {
    justify-content: stretch;
  }

  .start-button {
    width: 100%;
  }
}








questionnaire-close.ts

import {
  Component,
  EventEmitter,
  Input,
  Output
} from '@angular/core';

@Component({
  selector: 'app-questionnaire-close',
  standalone: true,
  templateUrl:
    './questionnaire-close.html',
  styleUrl:
    './questionnaire-close.css'
})
export class QuestionnaireClose {

  @Input()
  remainingCount = 0;

  @Input()
  saving = false;

  @Output()
  continueQuestionnaire =
    new EventEmitter<void>();

  @Output()
  saveAndClose =
    new EventEmitter<void>();

  continue(): void {

    if (this.saving) {
      return;
    }

    this.continueQuestionnaire.emit();
  }

  close(): void {

    if (this.saving) {
      return;
    }

    this.saveAndClose.emit();
  }
}









<div class="modal-backdrop">

  <div class="close-modal">

    <h2>
      More questionnaires remain
    </h2>

    <p>

      @if (remainingCount === 1) {

        There is 1 questionnaire left to complete.

      } @else {

        There are {{ remainingCount }}
        questionnaires left to complete.

      }

    </p>

    <p>
      Are you sure you want to save your current answers
      and close this window?
    </p>

    <div class="modal-actions">

      <button
        type="button"
        class="continue-button"
        [disabled]="saving"
        (click)="continue()"
      >
        Continue Questionnaire
      </button>

      <button
        type="button"
        class="close-button"
        [disabled]="saving"
        (click)="close()"
      >

        @if (saving) {

          Saving...

        } @else {

          Save & Close

        }

      </button>

    </div>

  </div>

</div>










.modal-backdrop {
  position: fixed;
  inset: 0;
  z-index: 1000;

  display: flex;
  align-items: center;
  justify-content: center;

  padding: 20px;

  background:
    rgba(0, 0, 0, 0.35);
}

.close-modal {
  width: 100%;
  max-width: 500px;

  background: #ffffff;

  border-radius: 10px;

  padding: 28px;

  box-sizing: border-box;

  box-shadow:
    0 14px 40px
    rgba(0, 0, 0, 0.2);
}

.close-modal h2 {
  margin: 0 0 14px;

  font-size: 21px;

  color: #263238;
}

.close-modal p {
  margin: 8px 0;

  font-size: 14px;
  line-height: 1.55;

  color: #546168;
}

.modal-actions {
  display: flex;
  justify-content: flex-end;

  gap: 10px;

  margin-top: 26px;
}

.continue-button,
.close-button {
  min-height: 40px;

  padding: 0 18px;

  border-radius: 6px;

  font-size: 14px;
  font-weight: 600;

  cursor: pointer;
}

.continue-button {
  background: #ffffff;

  color: #455a64;

  border: 1px solid #b8c3c8;
}

.continue-button:hover:not(:disabled) {
  background: #f4f7f8;
}

.close-button {
  border: none;

  background: #009688;

  color: #ffffff;
}

.close-button:hover:not(:disabled) {
  background: #00796b;
}

button:disabled {
  opacity: 0.6;

  cursor: not-allowed;
}












first-visit-questionnaire.ts



import {
  CommonModule
} from '@angular/common';

import {
  ChangeDetectorRef,
  Component,
  EventEmitter,
  Input,
  OnChanges,
  Output,
  SimpleChanges
} from '@angular/core';

import {
  FormsModule
} from '@angular/forms';

import {
  HttpClient
} from '@angular/common/http';

import {
  forkJoin
} from 'rxjs';

import {
  BaselineWho,
  REQUIRED_MESSAGE
} from '../history-questionnaire/history-questionnaire';

import {
  fillName,
  QuestionnaireLanguage,
  translate
} from '../legacy-i18n';

import {
  QuestionnaireProgressValue
} from '../questionnaire-progress.model';


interface Choice {
  value:
    string |
    number |
    null;

  label:
    string |
    null;
}


interface BaselineSection {
  fields:
    Record<
      string,
      string |
      number |
      null
    >;

  lists:
    Record<
      string,
      unknown[]
    >;

  version:
    string;
}


interface BaselinePage {
  baseline:
    BaselineSection;
}


type LabelledOption =
  string |
  {
    value:
      string;

    label:
      string |
      null;
  };


interface RawBaselineOptions {

  relationships:
    LabelledOption[];

  premature:
    LabelledOption[];

  birthStays:
    LabelledOption[];

  ages:
    LabelledOption[];

  learning:
    Choice[];
}


interface BaselineOptions {

  relationships:
    string[];

  premature:
    string[];

  birthStays:
    string[];

  ages:
    string[];

  learning:
    Choice[];
}


@Component({
  selector:
    'app-first-visit-questionnaire',

  standalone:
    true,

  imports: [
    CommonModule,
    FormsModule
  ],

  templateUrl:
    './first-visit-questionnaire.html',

  styleUrl:
    './first-visit-questionnaire.css'
})
export class FirstVisitQuestionnaire
implements OnChanges {

  @Input()
  patientId:
    number |
    null = null;


  @Input()
  firstName = '';


  @Input()
  hideActions =
    false;


  @Input()
  language:
    QuestionnaireLanguage =
      'en';


  @Output()
  saveComplete =
    new EventEmitter<void>();


  @Output()
  saveFailed =
    new EventEmitter<string>();


  @Output()
  baselineChange =
    new EventEmitter<BaselineWho>();


  @Output()
  progressChange =
    new EventEmitter<void>();


  loading =
    false;

  loadError =
    '';

  saving =
    false;

  saveError =
    '';

  saved =
    false;


  section:
    BaselineSection |
    null = null;


  options:
    BaselineOptions |
    null = null;


  private labels:
    Record<
      string,
      string
    > = {};


  constructor(
    private http:
      HttpClient,

    private cdr:
      ChangeDetectorRef
  ) {}


  ngOnChanges(
    changes:
      SimpleChanges
  ): void {

    if (
      !(
        changes['patientId'] ||
        changes['language']
      )
    ) {
      return;
    }


    this.saved =
      false;

    this.saveError =
      '';

    this.section =
      null;

    this.options =
      null;

    this.load();
  }


  text(
    key:
      string
  ): string {

    return fillName(
      translate(
        key,
        this.language
      ),
      this.firstName,
      this.language === 'sp'
        ? 'su niño(a)'
        : 'your child'
    );
  }


  lab(
    kind:
      string,

    value:
      string |
      number |
      null |
      undefined
  ): string {

    const key =
      String(
        value ?? ''
      );

    return (
      this.labels[
        `${kind}:${key}`
      ] ??
      key
    );
  }


  private base():
    string {

    return (
      `/api/patients/` +
      `${this.patientId}/` +
      `questionnaire-responses`
    );
  }


  private load():
    void {

    if (
      !this.patientId
    ) {
      return;
    }


    this.loading =
      true;

    this.loadError =
      '';


    forkJoin({

      page:
        this.http.get<BaselinePage>(
          this.base()
        ),

      options:
        this.http.get<RawBaselineOptions>(
          `${this.base()}/options`,
          {
            params: {
              language:
                this.language
            }
          }
        )

    })
    .subscribe({

      next:
        ({
          page,
          options
        }) => {

          this.section =
            page.baseline;


          this.labels =
            {};


          const values =
            (
              kind:
                string,

              list:
                LabelledOption[]
            ) =>
              list.map(
                item => {

                  if (
                    typeof item ===
                    'string'
                  ) {
                    return item;
                  }


                  if (
                    item.label
                  ) {

                    this.labels[
                      `${kind}:${item.value}`
                    ] =
                      item.label;
                  }


                  return (
                    item.value
                  );
                }
              );


          this.options = {

            relationships:
              values(
                'relationships',
                options.relationships ??
                []
              ),

            premature:
              values(
                'premature',
                options.premature ??
                []
              ),

            birthStays:
              values(
                'birthStays',
                options.birthStays ??
                []
              ),

            ages:
              values(
                'ages',
                options.ages ??
                []
              ),

            learning:
              options.learning ??
              []
          };


          this.loading =
            false;


          this.emitWho();


          this.cdr
            .markForCheck();


          this.progressChange
            .emit();
        },


      error:
        error => {

          console.error(
            'Unable to load First Visit History:',
            error
          );


          this.loading =
            false;


          this.loadError =
            'Unable to load this questionnaire.';


          this.cdr
            .markForCheck();
        }
    });
  }


  field(
    key:
      string
  ):
    string |
    number |
    null {

    return (
      this.section
        ?.fields[key] ??
      null
    );
  }


  setField(
    key:
      string,

    value:
      string |
      number |
      null
  ): void {

    if (
      !this.section
    ) {
      return;
    }


    this.section
      .fields[key] =
        value;


    if (
      key ===
      'historyRelationship'
    ) {
      this.emitWho();
    }


    this.progressChange
      .emit();
  }


  setPerson(
    name:
      string
  ): void {

    if (
      !this.section
    ) {
      return;
    }


    this.section
      .fields['historyPerson'] =
        name;


    this.section
      .fields['baselineDate'] =
        this.today();


    this.emitWho();


    this.progressChange
      .emit();
  }


  private emitWho():
    void {

    const fields =
      this.section
        ?.fields;


    this.baselineChange
      .emit({

        person:
          (
            fields
              ?.['historyPerson']
            as
              string |
              null
          ) ??
          null,

        relationship:
          (
            fields
              ?.['historyRelationship']
            as
              string |
              null
          ) ??
          null
      });
  }


  private today():
    string {

    const now =
      new Date();


    const month =
      String(
        now.getMonth() +
        1
      )
      .padStart(
        2,
        '0'
      );


    const day =
      String(
        now.getDate()
      )
      .padStart(
        2,
        '0'
      );


    return (
      `${now.getFullYear()}` +
      `-${month}` +
      `-${day}`
    );
  }


  getProgress():
    QuestionnaireProgressValue {

    const fields =
      this.section
        ?.fields ??
      {};


    const keys = [
      'historyPerson',
      'historyRelationship',
      'premature',
      'birthStay',
      'ageWalk',
      'ageTalk',
      'learning'
    ];


    const blank =
      (
        value:
          unknown
      ) =>
        value === null ||
        value === undefined ||
        value === '' ||
        value === '--';


    return {

      answered:
        keys.filter(
          key =>
            !blank(
              fields[key]
            )
        )
        .length,

      total:
        keys.length
    };
  }


  validate():
    string |
    null {

    const fields =
      this.section
        ?.fields;


    if (
      !fields
    ) {
      return null;
    }


    const keys = [
      'historyPerson',
      'historyRelationship',
      'premature',
      'birthStay',
      'ageWalk',
      'ageTalk',
      'learning'
    ];


    const blank =
      (
        value:
          unknown
      ) =>
        value === null ||
        value === undefined ||
        value === '' ||
        value === '--';


    return (
      keys.some(
        key =>
          blank(
            fields[key]
          )
      )
        ? translate(
            REQUIRED_MESSAGE,
            this.language
          )
        : null
    );
  }


  save():
    void {

    if (
      !this.patientId ||
      !this.section
    ) {

      this.saveComplete
        .emit();

      return;
    }


    for (
      const key of [
        'historyPerson',
        'historyRelationship',
        'premature',
        'birthStay',
        'ageWalk',
        'ageTalk'
      ]
    ) {

      if (
        this.section
          .fields[key] ===
          null ||
        this.section
          .fields[key] ===
          undefined
      ) {

        this.section
          .fields[key] =
            '';
      }
    }


    if (
      this.section
        .fields['learning'] ===
        '' ||
      this.section
        .fields['learning'] ===
        undefined
    ) {

      this.section
        .fields['learning'] =
          null;
    }


    this.saving =
      true;

    this.saveError =
      '';


    this.http
      .put<void>(
        `${this.base()}/baseline`,
        this.section
      )
      .subscribe({

        next:
          () => {

            this.saving =
              false;

            this.saved =
              true;

            this.cdr
              .markForCheck();

            this.saveComplete
              .emit();
          },


        error:
          error => {

            console.error(
              'Unable to save First Visit History:',
              error
            );


            this.saving =
              false;


            this.saveError =
              'Unable to save. Please try again.';


            this.cdr
              .markForCheck();


            this.saveFailed
              .emit(
                this.saveError
              );
          }
      });
  }
}





<div class="history-form">

  @if (loading) {

    <div class="loading-text">
      Loading…
    </div>

  } @else if (loadError) {

    <div class="questionnaire-error">
      {{ loadError }}
    </div>

  } @else if (
    section &&
    options
  ) {

    @if (saved) {

      <div class="submitted-banner">
        First Visit History saved.
      </div>

    }


    @if (
      saveError &&
      !hideActions
    ) {

      <div class="questionnaire-error">
        {{ saveError }}
      </div>

    }


    <div class="history-body">


      <section class="history-page">

        <h2 class="page-heading">
          {{ text('Please tell us about you') }}
        </h2>


        <div class="field-row">

          <label
            class="field-label"
            for="historyPerson"
          >
            {{ text('Your Name:') }}
          </label>

          <input
            id="historyPerson"
            type="text"
            size="30"
            [ngModel]="field('historyPerson')"
            (ngModelChange)="setPerson($event)"
          />

        </div>


        <div class="field-row">

          <label
            class="field-label"
            for="historyRelationship"
          >

            {{
              text(
                'What is your relationship to [name]?'
              )
            }}

            <span class="field-hint">
              {{
                text(
                  'Choose from the list'
                )
              }}
            </span>

          </label>


          <select
            id="historyRelationship"
            [ngModel]="field('historyRelationship')"
            (ngModelChange)="
              setField(
                'historyRelationship',
                $event
              )
            "
          >

            <option
              [ngValue]="null"
            >
            </option>


            @for (
              choice of options.relationships;
              track choice
            ) {

              <option
                [ngValue]="choice"
              >
                {{
                  lab(
                    'relationships',
                    choice
                  )
                }}
              </option>

            }

          </select>

        </div>

      </section>


      <section class="history-page">

        <h2 class="page-heading">
          {{ text("[name]'s Birth") }}
        </h2>


        <div class="field-row">

          <label
            class="field-label"
            for="premature"
          >
            {{
              text(
                'How premature was [name]?'
              )
            }}
          </label>


          <select
            id="premature"
            [ngModel]="field('premature')"
            (ngModelChange)="
              setField(
                'premature',
                $event
              )
            "
          >

            <option
              [ngValue]="null"
            >
            </option>


            @for (
              choice of options.premature;
              track choice
            ) {

              <option
                [ngValue]="choice"
              >
                {{
                  lab(
                    'premature',
                    choice
                  )
                }}
              </option>

            }

          </select>

        </div>


        <div class="field-row">

          <label
            class="field-label"
            for="birthStay"
          >
            {{
              text(
                'How much extended hospitalization did [name] require after birth?'
              )
            }}
          </label>


          <select
            id="birthStay"
            [ngModel]="field('birthStay')"
            (ngModelChange)="
              setField(
                'birthStay',
                $event
              )
            "
          >

            <option
              [ngValue]="null"
            >
            </option>


            @for (
              choice of options.birthStays;
              track choice
            ) {

              <option
                [ngValue]="choice"
              >
                {{
                  lab(
                    'birthStays',
                    choice
                  )
                }}
              </option>

            }

          </select>

        </div>

      </section>


      <section class="history-page">

        <h2 class="page-heading">
          {{
            text(
              "[name]'s Walking History"
            )
          }}
        </h2>


        <div class="field-row">

          <label
            class="field-label"
            for="ageWalk"
          >
            {{
              text(
                'What age did [name] first walk?'
              )
            }}
          </label>


          <select
            id="ageWalk"
            [ngModel]="field('ageWalk')"
            (ngModelChange)="
              setField(
                'ageWalk',
                $event
              )
            "
          >

            <option
              [ngValue]="null"
            >
            </option>


            @for (
              choice of options.ages;
              track choice
            ) {

              <option
                [ngValue]="choice"
              >
                {{
                  lab(
                    'ages',
                    choice
                  )
                }}
              </option>

            }

          </select>

        </div>


        <div class="field-row">

          <label
            class="field-label"
            for="ageTalk"
          >
            {{
              text(
                'What age did [name] start talking?'
              )
            }}
          </label>


          <select
            id="ageTalk"
            [ngModel]="field('ageTalk')"
            (ngModelChange)="
              setField(
                'ageTalk',
                $event
              )
            "
          >

            <option
              [ngValue]="null"
            >
            </option>


            @for (
              choice of options.ages;
              track choice
            ) {

              <option
                [ngValue]="choice"
              >
                {{
                  lab(
                    'ages',
                    choice
                  )
                }}
              </option>

            }

          </select>

        </div>

      </section>


      <section class="history-page">

        <h2 class="page-heading">
          {{ text('Learning') }}
        </h2>


        <div class="field-row">

          <label
            class="field-label"
            for="learning"
          >
            {{
              text(
                'How do you think [name] is able to learn compared to other children of the same age?'
              )
            }}
          </label>


          <select
            id="learning"
            [ngModel]="field('learning')"
            (ngModelChange)="
              setField(
                'learning',
                $event
              )
            "
          >

            <option
              [ngValue]="null"
            >
            </option>


            @for (
              choice of options.learning;
              track choice.value
            ) {

              <option
                [ngValue]="choice.value"
              >
                {{ choice.label }}
              </option>

            }

          </select>

        </div>

      </section>

    </div>


    @if (!hideActions) {

      <div class="history-actions">

        <button
          type="button"
          class="submit-btn"
          [disabled]="saving"
          (click)="save()"
        >
          {{
            saving
              ? 'Saving...'
              : (
                  saved
                    ? 'Save Again'
                    : 'Save'
                )
          }}
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
  width: 100%;
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

  box-shadow:
    0 0 0 3px
    rgba(38, 156, 150, 0.12);
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

  box-shadow:
    0 1px 3px
    rgba(31, 132, 127, 0.25);

  transition:
    background-color 0.15s ease,
    box-shadow 0.15s ease;
}

.submit-btn:hover:not(:disabled) {
  background: #1f847f;

  box-shadow:
    0 2px 6px
    rgba(31, 132, 127, 0.3);
}

.submit-btn:disabled {
  background: #b7d4d2;

  border-color: #b7d4d2;

  box-shadow: none;

  cursor: not-allowed;
}









history-questionnaire.ts




import { CommonModule } from '@angular/common';
import {
  ChangeDetectorRef,
  Component,
  EventEmitter,
  Input,
  OnChanges,
  Output,
  SimpleChanges
} from '@angular/core';
import { FormsModule } from '@angular/forms';
import { HttpClient } from '@angular/common/http';
import { forkJoin, from, of } from 'rxjs';
import {
  fillName,
  QuestionnaireLanguage,
  translate
} from '../legacy-i18n';
import {
  QuestionnaireProgressValue
} from '../questionnaire-progress.model';
import {
  catchError,
  concatMap,
  map,
  switchMap,
  tap,
  toArray
} from 'rxjs/operators';

import {
  PatientHistoryData,
  HealthHistoryEntry,
  HistoryConditionOption,
  HealthConditionEntry,
  ConditionOption,
  BotoxHistory,
  SideOption
} from '../../patients/history/history';

type Row =
  Record<
    string,
    string |
    number |
    null
  >;

interface Choice {
  value:
    string |
    number |
    null;

  label:
    string |
    null;

  description?:
    string |
    null;
}

interface Section {
  fields:
    Record<
      string,
      string |
      number |
      null
    >;

  lists:
    Record<
      string,
      Row[]
    >;

  version:
    string;
}

interface VisitResponses {
  visitId:
    number;

  visitDate:
    string |
    null;

  items:
    Section;

  medications:
    Section;

  details:
    Section;
}

type LabelledOption =
  string |
  {
    value:
      string;

    label:
      string |
      null;
  };

interface NormalizedOptions {
  relationships:
    string[];

  walkingSupport:
    Choice[];

  walkingChanges:
    Choice[];

  fms:
    Choice[];

  devices:
    string[];

  gaitConcerns:
    Choice[];

  pain:
    string[];
}

interface HistoryOptions {
  relationships:
    LabelledOption[];

  walkingSupport:
    Choice[];

  walkingChanges:
    Choice[];

  fms:
    Choice[];

  devices:
    LabelledOption[];

  gaitConcerns:
    Choice[];

  pain:
    LabelledOption[];
}

export const REQUIRED_MESSAGE =
  'Please fill in all answers before moving to the next question.';

export interface BaselineWho {
  person:
    string |
    null;

  relationship:
    string |
    null;
}

@Component({
  selector:
    'app-history-questionnaire',

  standalone:
    true,

  imports: [
    CommonModule,
    FormsModule
  ],

  templateUrl:
    './history-questionnaire.html',

  styleUrl:
    './history-questionnaire.css'
})
export class HistoryQuestionnaire
implements OnChanges {

  @Input()
  patientId:
    number |
    null = null;

  @Input()
  visitId:
    number |
    null = null;

  @Input()
  firstName = '';

  @Input()
  hideActions =
    false;

  @Input()
  language:
    QuestionnaireLanguage =
      'en';

  @Input()
  showWho =
    true;

  @Input()
  showGeneral =
    true;

  @Input()
  showGait =
    true;

  @Input()
  showConcerns =
    true;

  @Input()
  copyWho:
    BaselineWho |
    null = null;

  @Output()
  saveComplete =
    new EventEmitter<void>();

  @Output()
  saveFailed =
    new EventEmitter<string>();

  @Output()
  progressChange =
    new EventEmitter<void>();


  frequencyChoices:
    {
      value:
        string;

      label:
        string;
    }[] = [

      {
        value: '4',
        label: 'None'
      },

      {
        value: '3',
        label:
          'About once per month'
      },

      {
        value: '2',
        label:
          'Two or three times per month'
      },

      {
        value: '1',
        label:
          'One or more times per week'
      }
    ];


  readonly therapyGroups = [

    {
      title:
        'PHYSICAL THERAPY',

      fields: [

        {
          key:
            'hospitalPt',

          label:
            'Hospital'
        },

        {
          key:
            'clinicPt',

          label:
            'Clinic'
        },

        {
          key:
            'schoolPt',

          label:
            'School'
        },

        {
          key:
            'homePt',

          label:
            'Home'
        }
      ]
    },

    {
      title:
        'OCCUPATIONAL THERAPY',

      fields: [

        {
          key:
            'hospitalOt',

          label:
            'Hospital'
        },

        {
          key:
            'clinicOt',

          label:
            'Clinic'
        },

        {
          key:
            'schoolOt',

          label:
            'School'
        },

        {
          key:
            'homeOt',

          label:
            'Home'
        }
      ]
    }
  ];


  readonly deviceSideOptions = [

    {
      value:
        'Both',

      label:
        'Both'
    },

    {
      value:
        'R',

      label:
        'R only'
    },

    {
      value:
        'L',

      label:
        'L only'
    }
  ];


  readonly fmsQuestions = [

    {
      key:
        'fms5Id',

      text:
        'How does [name] move around for short distances in the house?'
    },

    {
      key:
        'fms50Id',

      text:
        'How does [name] move around in and between classes at school?'
    },

    {
      key:
        'fms500Id',

      text:
        'How does [name] move around for long distances such as the shopping center?'
    }
  ];


  loading =
    false;

  loadError =
    '';

  saving =
    false;

  saveError =
    '';

  saved =
    false;


  items:
    Section |
    null = null;

  details:
    Section |
    null = null;

  medications:
    Section |
    null = null;

  options:
    NormalizedOptions |
    null = null;


  healthHistory:
    HealthHistoryEntry[] = [];

  healthConditions:
    HealthConditionEntry[] = [];

  botox:
    BotoxHistory[] = [];


  historyConditionOptions:
    HistoryConditionOption[] = [];

  ageOptions:
    string[] = [];

  conditionOptions:
    ConditionOption[] = [];

  bodyLocationOptions:
    string[] = [];

  botoxSideOptions:
    SideOption[] = [];

  seizureMedOptions:
    string[] = [];


  newHistoryConditionCode =
    '';

  newHistoryAge =
    '';

  historyError =
    '';

  historySaving =
    false;


  conditionError =
    '';

  confirmingConditionId:
    number |
    null = null;


  showCustomSeizureMed =
    false;

  customSeizureMed =
    '';


  showBotoxForm =
    false;

  newBotox = {
    bodyLocation: '',
    date: '',
    side: '',
    facility: '',
    physician: ''
  };

  botoxSaving =
    false;

  botoxError =
    '';


  showCustomDevice =
    false;

  customDevice =
    '';

  pendingDeviceSide:
    Record<
      string,
      string
    > = {};


  constructor(
    private http:
      HttpClient,

    private cdr:
      ChangeDetectorRef
  ) {}


  ngOnChanges(
    changes:
      SimpleChanges
  ): void {

    if (
      !(
        changes['patientId'] ||
        changes['visitId'] ||
        changes['language'] ||
        changes['showWho'] ||
        changes['showGeneral'] ||
        changes['showGait'] ||
        changes['showConcerns']
      )
    ) {
      return;
    }


    this.saved =
      false;

    this.saveError =
      '';

    this.items =
      null;

    this.details =
      null;

    this.medications =
      null;

    this.options =
      null;

    this.healthHistory =
      [];

    this.healthConditions =
      [];

    this.botox =
      [];

    this.newHistoryConditionCode =
      '';

    this.newHistoryAge =
      '';

    this.historyError =
      '';

    this.conditionError =
      '';

    this.pendingConditions =
      new Set<string>();

    this.confirmingConditionId =
      null;

    this.pendingRemoval =
      null;

    this.showCustomSeizureMed =
      false;

    this.customSeizureMed =
      '';

    this.showBotoxForm =
      false;

    this.newBotox = {
      bodyLocation: '',
      date: '',
      side: '',
      facility: '',
      physician: ''
    };

    this.botoxError =
      '';

    this.showCustomDevice =
      false;

    this.customDevice =
      '';

    this.pendingDeviceSide =
      {};

    this.load();
  }


  get langParams():
    {
      language:
        string;
    } {

    return {
      language:
        this.language
    };
  }


  private labels:
    Record<
      'relationship' |
      'age' |
      'device' |
      'pain' |
      'bodyLoc',
      Record<
        string,
        string
      >
    > = {

      relationship: {},
      age: {},
      device: {},
      pain: {},
      bodyLoc: {}
    };


  private normalize(
    kind:
      'relationship' |
      'age' |
      'device' |
      'pain' |
      'bodyLoc',

    list:
      LabelledOption[] |
      undefined
  ): string[] {

    return (
      list ??
      []
    )
    .map(
      item => {

        if (
          typeof item ===
          'string'
        ) {
          return item;
        }


        if (
          item.label
        ) {

          this.labels[
            kind
          ][
            item.value
          ] =
            item.label;
        }


        return (
          item.value
        );
      }
    );
  }


  lab(
    kind:
      'relationship' |
      'age' |
      'device' |
      'pain' |
      'bodyLoc',

    value:
      string |
      number |
      null |
      undefined
  ): string {

    const key =
      String(
        value ??
        ''
      );

    return (
      this.labels[
        kind
      ][key] ??
      key
    );
  }


  conditionName(
    code:
      string |
      null |
      undefined,

    fallback:
      string |
      null |
      undefined
  ): string {

    const found =
      this.conditionOptions
        .find(
          option =>
            option.code ===
            code
        );


    return (
      found?.name ??
      fallback ??
      ''
    );
  }


  historyConditionName(
    code:
      string |
      null |
      undefined,

    fallback:
      string |
      null |
      undefined
  ): string {

    const found =
      this.historyConditionOptions
        .find(
          option =>
            option.code ===
            code
        );


    return (
      found?.name ??
      fallback ??
      ''
    );
  }


  walkingChangeLabel(
    choice:
      Choice
  ): string {

    const english:
      Record<
        string,
        string
      > = {

      'No Change':
        'No Change',

      'Much Better':
        'Walks Much Better',

      'Little Better':
        'Walks a Little Better',

      'Little Worse':
        'Walks a Little Worse',

      'Much Worse':
        'Walks Much Worse'
    };


    return (
      choice.label &&
      this.language === 'en'

        ? choice.label

        : this.text(
            english[
              String(
                choice.value
              )
            ] ??
            String(
              choice.label ??
              choice.value ??
              ''
            )
          )
    );
  }


  text(
    key:
      string
  ): string {

    return fillName(
      translate(
        key,
        this.language
      ),

      this.firstName,

      this.language === 'sp'
        ? 'su niño(a)'
        : 'your child'
    );
  }


  pendingRemoval:
    {
      kind:
        'device' |
        'seizure' |
        'concern';

      key:
        string |
        number |
        null;

      label:
        string;
    } |
    null = null;


  askRemove(
    kind:
      'device' |
      'seizure' |
      'concern',

    key:
      string |
      number |
      null,

    label:
      string
  ): void {

    this.pendingRemoval = {
      kind,
      key,
      label
    };
  }


  cancelRemove():
    void {

    this.pendingRemoval =
      null;
  }


  removalText(
    part:
      'title' |
      'question' |
      'confirm' |
      'cancel'
  ): string {

    const kind =
      this.pendingRemoval?.kind;


    const englishOnly =
      kind === 'concern' &&
      part !== 'cancel';


    const lang:
      QuestionnaireLanguage =
        englishOnly
          ? 'en'
          : this.language;


    const key =
      part === 'title'

        ? (
            kind === 'device'
              ? 'Confirm Device Deletion'

              : kind === 'seizure'
                ? 'Confirm Medication Deletion'

                : 'Confirm Gait Concern Deletion'
          )

        : part === 'question'

          ? 'Are you sure you want to remove the following entry from your list?'

          : part === 'confirm'

            ? (
                kind === 'concern'
                  ? 'Yes - Delete this from my List'

                  : 'Yes - Delete this from my List#dot'
              )

            : 'Do Not Delete this Item';


    return translate(
      key,
      lang
    );
  }


  confirmRemove():
    void {

    const removal =
      this.pendingRemoval;


    this.pendingRemoval =
      null;


    if (!removal) {
      return;
    }


    if (
      removal.kind ===
      'device'
    ) {

      this.removeDevice(
        String(
          removal.key
        )
      );

    } else if (
      removal.kind ===
      'seizure'
    ) {

      this.removeSeizureMed(
        String(
          removal.key
        )
      );

    } else {

      this.removeGaitConcern(
        removal.key
      );
    }


    this.progressChange
      .emit();
  }


  private base():
    string {

    return (
      `/api/patients/` +
      `${this.patientId}/` +
      `questionnaire-responses`
    );
  }


  private patientBase():
    string {

    return (
      `/api/patients/` +
      `${this.patientId}`
    );
  }


  private load():
    void {

    if (
      !this.patientId ||
      !this.visitId
    ) {
      return;
    }


    this.loading =
      true;

    this.loadError =
      '';


    forkJoin({

      visit:
        this.http.get<VisitResponses>(
          `${this.base()}/visits/${this.visitId}`
        ),

      options:
        this.http.get<HistoryOptions>(
          `${this.base()}/options`,
          {
            params:
              this.langParams
          }
        ),

      patientHistory:
        this.http.get<PatientHistoryData>(
          `${this.patientBase()}/history`
        ),

      historyConditions:
        this.http.get<
          HistoryConditionOption[]
        >(
          '/api/reference/history-conditions',
          {
            params:
              this.langParams
          }
        ),

      ages:
        this.http.get<
          LabelledOption[]
        >(
          '/api/reference/ages',
          {
            params:
              this.langParams
          }
        ),

      conditions:
        this.http.get<
          ConditionOption[]
        >(
          '/api/reference/conditions',
          {
            params:
              this.langParams
          }
        ),

      bodyLocations:
        this.http.get<
          LabelledOption[]
        >(
          '/api/reference/body-locations',
          {
            params:
              this.langParams
          }
        ),

      botoxSides:
        this.http.get<
          SideOption[]
        >(
          '/api/reference/botox-sides',
          {
            params:
              this.langParams
          }
        ),

      seizureMedOptions:
        this.http.get<
          string[]
        >(
          '/api/reference/seizure-meds'
        ),

      frequencies:
        this.http.get<
          {
            value:
              string;

            label:
              string |
              null;
          }[]
        >(
          '/api/reference/frequencies',
          {
            params:
              this.langParams
          }
        )

    })
    .subscribe({

      next:
        result => {

          this.items =
            result.visit.items;

          this.details =
            result.visit.details;

          this.medications =
            result.visit.medications;


          this.labels = {
            relationship: {},
            age: {},
            device: {},
            pain: {},
            bodyLoc: {}
          };


          this.options = {

            ...result.options,

            relationships:
              this.normalize(
                'relationship',
                result.options.relationships
              ),

            devices:
              this.normalize(
                'device',
                result.options.devices
              ),

            pain:
              this.normalize(
                'pain',
                result.options.pain
              )
          };


          this.healthHistory =
            result.patientHistory
              .healthHistory ??
            [];

          this.healthConditions =
            result.patientHistory
              .healthConditions ??
            [];

          this.botox =
            result.patientHistory
              .botox ??
            [];


          this.historyConditionOptions =
            result.historyConditions ??
            [];

          this.ageOptions =
            this.normalize(
              'age',
              result.ages
            );

          this.conditionOptions =
            result.conditions ??
            [];

          this.bodyLocationOptions =
            this.normalize(
              'bodyLoc',
              result.bodyLocations
            );

          this.botoxSideOptions =
            result.botoxSides ??
            [];

          this.seizureMedOptions =
            result.seizureMedOptions ??
            [];


          if (
            result.frequencies
              ?.length
          ) {

            this.frequencyChoices =
              result.frequencies
                .map(
                  item => ({
                    value:
                      String(
                        item.value
                      ),

                    label:
                      item.label ??
                      String(
                        item.value
                      )
                  })
                );
          }


          this.applyLegacyDefaults();


          this.loading =
            false;


          this.cdr
            .markForCheck();


          this.progressChange
            .emit();
        },


      error:
        error => {

          console.error(
            'Unable to load History:',
            error
          );


          this.loading =
            false;


          this.loadError =
            'Unable to load this questionnaire.';


          this.cdr
            .markForCheck();
        }
    });
  }


  private applyLegacyDefaults():
    void {

    if (
      !this.details
    ) {
      return;
    }


    for (
      const group of
        this.therapyGroups
    ) {

      for (
        const field of
          group.fields
      ) {

        const current =
          this.details
            .fields[
              field.key
            ];


        if (
          current === null ||
          current === undefined ||
          current === ''
        ) {

          this.details
            .fields[
              field.key
            ] =
              '4';
        }
      }
    }
  }


  setItem(
    key:
      string,

    value:
      string |
      number |
      null
  ): void {

    if (
      this.items
    ) {

      this.items
        .fields[
          key
        ] =
          value;


      this.progressChange
        .emit();
    }
  }


  private reloadPatientHistory():
    void {

    if (
      !this.patientId
    ) {
      return;
    }


    this.http
      .get<PatientHistoryData>(
        `${this.patientBase()}/history`
      )
      .subscribe({

        next:
          history => {

            this.healthHistory =
              history.healthHistory ??
              [];

            this.healthConditions =
              history.healthConditions ??
              [];

            this.botox =
              history.botox ??
              [];


            this.cdr
              .markForCheck();


            this.progressChange
              .emit();
          },


        error:
          error => {

            console.error(
              'Unable to reload patient history:',
              error
            );


            this.cdr
              .markForCheck();
          }
      });
  }


  addHealthHistory():
    void {

    if (
      !this.patientId ||
      !this.newHistoryConditionCode ||
      !this.newHistoryAge
    ) {

      this.historyError =
        !this.newHistoryAge

          ? translate(
              REQUIRED_MESSAGE,
              this.language
            )

          : 'Please select a condition before moving to the next question.';


      return;
    }


    this.historySaving =
      true;

    this.historyError =
      '';


    this.http
      .post<HealthHistoryEntry>(
        `${this.patientBase()}/history/health-history`,
        {
          age:
            this.newHistoryAge,

          conditionCode:
            this.newHistoryConditionCode
        }
      )
      .subscribe({

        next:
          () => {

            this.newHistoryConditionCode =
              '';

            this.newHistoryAge =
              '';

            this.historySaving =
              false;


            this.reloadPatientHistory();
          },


        error:
          error => {

            console.error(
              'Unable to add health history item:',
              error
            );


            this.historySaving =
              false;


            this.historyError =
              'Unable to add. Please try again.';


            this.cdr
              .markForCheck();
          }
      });
  }


  availableConditionOptions():
    ConditionOption[] {

    const existing =
      new Set(
        this.healthConditions
          .map(
            item =>
              (
                item.conditionCode ??
                ''
              )
              .trim()
              .toLowerCase()
          )
      );


    return (
      this.conditionOptions
        .filter(
          option =>
            !existing.has(
              option.code
                .trim()
                .toLowerCase()
            )
        )
    );
  }


  pendingConditions =
    new Set<string>();


  isConditionPending(
    code:
      string
  ): boolean {

    return (
      this.pendingConditions
        .has(code)
    );
  }


  togglePendingCondition(
    code:
      string
  ): void {

    if (
      this.pendingConditions
        .has(code)
    ) {

      this.pendingConditions
        .delete(code);

    } else {

      this.pendingConditions
        .add(code);
    }


    this.progressChange
      .emit();
  }


  askDeleteCondition(
    id:
      number
  ): void {

    this.confirmingConditionId =
      id;
  }


  cancelDeleteCondition():
    void {

    this.confirmingConditionId =
      null;
  }


  get confirmingCondition():
    HealthConditionEntry |
    undefined {

    return (
      this.healthConditions
        .find(
          entry =>
            entry.id ===
            this.confirmingConditionId
        )
    );
  }


  confirmDeleteCondition():
    void {

    const id =
      this.confirmingConditionId;


    if (
      !this.patientId ||
      id === null
    ) {
      return;
    }


    this.http
      .delete<void>(
        `${this.patientBase()}/history/health-conditions/${id}`
      )
      .subscribe({

        next:
          () => {

            this.confirmingConditionId =
              null;


            this.reloadPatientHistory();
          },


        error:
          error => {

            console.error(
              'Unable to remove health condition:',
              error
            );


            this.conditionError =
              'Unable to remove. Please try again.';


            this.cdr
              .markForCheck();
          }
      });
  }


  get hasSeizuresCondition():
    boolean {

    const recorded =
      this.healthConditions
        .some(
          entry => {

            const text =
              `${
                entry.conditionCode ??
                ''
              } ${
                entry.conditionDescription ??
                ''
              }`
              .toLowerCase();


            return (
              entry.conditionCode ===
                'SEIZ' ||
              text.includes(
                'seizure'
              )
            );
          }
        );


    return (
      recorded ||
      this.pendingConditions
        .has(
          'SEIZ'
        )
    );
  }


  get seizureMedRows():
    Row[] {

    return (
      this.medications
        ?.lists[
          'medications'
        ] ??
      []
    );
  }


  rowName(
    row:
      Row
  ): string {

    return String(
      row['name'] ??
      ''
    );
  }


  availableSeizureMeds():
    string[] {

    const added =
      new Set(
        this.seizureMedRows
          .map(
            row =>
              this.rowName(
                row
              )
              .toLowerCase()
          )
      );


    return (
      this.seizureMedOptions
        .filter(
          name =>
            !added.has(
              name.toLowerCase()
            )
        )
    );
  }


  addSeizureMed(
    name:
      string
  ): void {

    const trimmed =
      name.trim();


    if (
      !trimmed ||
      !this.medications
    ) {
      return;
    }


    const rows =
      this.medications
        .lists[
          'medications'
        ] ??
      [];


    this.medications
      .lists[
        'medications'
      ] = [
        ...rows,

        {
          id: null,
          name: trimmed
        }
      ];


    this.progressChange
      .emit();
  }


  addCustomSeizureMed():
    void {

    this.addSeizureMed(
      this.customSeizureMed
    );


    this.customSeizureMed =
      '';

    this.showCustomSeizureMed =
      false;
  }


  removeSeizureMed(
    name:
      string
  ): void {

    if (
      !this.medications
    ) {
      return;
    }


    const rows =
      this.medications
        .lists[
          'medications'
        ] ??
      [];


    this.medications
      .lists[
        'medications'
      ] =
        rows.filter(
          row =>
            row['name'] !==
            name
        );


    this.progressChange
      .emit();
  }


  addBotox():
    void {

    if (
      !this.patientId
    ) {
      return;
    }


    this.botoxSaving =
      true;

    this.botoxError =
      '';


    this.http
      .post<BotoxHistory>(
        `${this.patientBase()}/history/botox`,
        this.newBotox
      )
      .subscribe({

        next:
          () => {

            this.newBotox = {
              bodyLocation: '',
              date: '',
              side: '',
              facility: '',
              physician: ''
            };


            this.botoxSaving =
              false;

            this.showBotoxForm =
              false;


            this.reloadPatientHistory();
          },


        error:
          error => {

            console.error(
              'Unable to add botox shot:',
              error
            );


            this.botoxSaving =
              false;


            this.botoxError =
              'Unable to add. Please try again.';


            this.cdr
              .markForCheck();
          }
      });
  }


  get deviceRows():
    Row[] {

    return (
      this.details
        ?.lists[
          'devices'
        ] ??
      []
    );
  }


  deviceHasSide(
    name:
      string
  ): boolean {

    return (
      name === 'Crutch' ||
      name === 'Cane'
    );
  }


  availableDevices():
    string[] {

    const added =
      new Set(
        this.deviceRows
          .map(
            row =>
              String(
                row['name'] ??
                ''
              )
          )
      );


    return (
      this.options
        ?.devices ??
      []
    )
    .filter(
      name =>
        !added.has(
          name
        )
    );
  }


  pendingSide(
    name:
      string
  ): string {

    return (
      this.pendingDeviceSide[
        name
      ] ??
      'Both'
    );
  }


  setPendingSide(
    name:
      string,

    side:
      string
  ): void {

    this.pendingDeviceSide[
      name
    ] =
      side;
  }


  addDevice(
    name:
      string
  ): void {

    if (
      !this.details
    ) {
      return;
    }


    const rows =
      this.details
        .lists[
          'devices'
        ] ??
      [];


    this.details
      .lists[
        'devices'
      ] = [
        ...rows,

        {
          id: null,
          name,

          side:
            this.deviceHasSide(
              name
            )
              ? this.pendingSide(
                  name
                )
              : null
        }
      ];


    this.progressChange
      .emit();
  }


  addCustomDevice():
    void {

    const name =
      this.customDevice
        .trim();


    if (
      name &&
      this.details
    ) {

      const rows =
        this.details
          .lists[
            'devices'
          ] ??
        [];


      this.details
        .lists[
          'devices'
        ] = [
          ...rows,

          {
            id: null,
            name,
            side: null
          }
        ];
    }


    this.customDevice =
      '';

    this.showCustomDevice =
      false;


    this.progressChange
      .emit();
  }


  removeDevice(
    name:
      string
  ): void {

    if (
      !this.details
    ) {
      return;
    }


    const rows =
      this.details
        .lists[
          'devices'
        ] ??
      [];


    this.details
      .lists[
        'devices'
      ] =
        rows.filter(
          row =>
            row['name'] !==
            name
        );


    this.progressChange
      .emit();
  }


  private normalizeSide(
    value:
      string |
      number |
      null |
      undefined
  ): string |
     null {

    const side =
      String(
        value ??
        ''
      );


    if (!side) {
      return null;
    }


    const lower =
      side.toLowerCase();


    return (
      lower === 'left'
        ? 'L'

        : lower === 'right'
          ? 'R'

          : side
    );
  }


  painSideChoices(
    bodyPart:
      string
  ):
    {
      value:
        string;

      label:
        string;
    }[] {

    return (
      bodyPart === 'Back'

        ? [

            {
              value: 'Upper',
              label:
                this.text(
                  'Upper'
                )
            },

            {
              value: 'Lower',
              label:
                this.text(
                  'Lower'
                )
            },

            {
              value: 'None',
              label:
                this.text(
                  'None'
                )
            }
          ]

        : [

            {
              value: 'L',
              label:
                this.text(
                  'Left#pain'
                )
            },

            {
              value: 'R',
              label:
                this.text(
                  'Right#pain'
                )
            },

            {
              value: 'Both',
              label:
                this.text(
                  'Both#pain'
                )
            },

            {
              value: 'None',
              label:
                this.text(
                  'None'
                )
            }
          ]
    );
  }


  painSide(
    bodyPart:
      string
  ): string {

    const row =
      this.details
        ?.lists[
          'pain'
        ]
        ?.find(
          row =>
            row['bodyPart'] ===
            bodyPart
        );


    return (
      row

        ? (
            this.normalizeSide(
              row['side']
            ) ??
            'None'
          )

        : 'None'
    );
  }


  setPainSide(
    bodyPart:
      string,

    side:
      string
  ): void {

    if (
      !this.details
    ) {
      return;
    }


    const rows =
      this.details
        .lists[
          'pain'
        ] ??
      [];


    const others =
      rows.filter(
        row =>
          row['bodyPart'] !==
          bodyPart
      );


    if (
      side === 'None'
    ) {

      this.details
        .lists[
          'pain'
        ] =
          others;


      this.progressChange
        .emit();

      return;
    }


    const existing =
      rows.find(
        row =>
          row['bodyPart'] ===
          bodyPart
      );


    this.details
      .lists[
        'pain'
      ] = [
        ...others,

        {
          id:
            existing
              ? existing['id']
              : null,

          bodyPart,
          side
        }
      ];


    this.progressChange
      .emit();
  }


  get gaitConcernRows():
    Row[] {

    return (
      this.details
        ?.lists[
          'gaitConcerns'
        ] ??
      []
    );
  }


  gaitConcernLabel(
    code:
      string |
      number |
      null
  ): string {

    const choice =
      (
        this.options
          ?.gaitConcerns ??
        []
      )
      .find(
        item =>
          item.value ===
          code
      );


    return (
      choice?.label ??
      String(
        code ??
        ''
      )
    );
  }


  availableGaitConcerns():
    Choice[] {

    const added =
      new Set(
        this.gaitConcernRows
          .map(
            row =>
              row['code']
          )
      );


    return (
      this.options
        ?.gaitConcerns ??
      []
    )
    .filter(
      choice =>
        !added.has(
          choice.value
        )
    );
  }


  addGaitConcern(
    code:
      string |
      number |
      null
  ): void {

    if (
      !this.details
    ) {
      return;
    }


    const rows =
      this.details
        .lists[
          'gaitConcerns'
        ] ??
      [];


    this.details
      .lists[
        'gaitConcerns'
      ] = [
        ...rows,

        {
          id: null,
          code
        }
      ];


    this.progressChange
      .emit();
  }


  removeGaitConcern(
    code:
      string |
      number |
      null
  ): void {

    if (
      !this.details
    ) {
      return;
    }


    const rows =
      this.details
        .lists[
          'gaitConcerns'
        ] ??
      [];


    this.details
      .lists[
        'gaitConcerns'
      ] =
        rows.filter(
          row =>
            row['code'] !==
            code
        );


    this.progressChange
      .emit();
  }


  getProgress():
    QuestionnaireProgressValue {

    const fields =
      this.items
        ?.fields ??
      {};


    const blank =
      (
        value:
          unknown
      ) =>
        value === null ||
        value === undefined ||
        value === '' ||
        value === '--';


    const keys:
      string[] = [];


    if (
      this.showWho
    ) {

      keys.push(
        'reportingPerson',
        'relationship'
      );
    }


    if (
      this.showGait
    ) {

      keys.push(
        'walkingSupportCode',
        'fms5Id',
        'fms50Id',
        'fms500Id',
        'walkingChange'
      );
    }


    return {

      answered:
        keys.filter(
          key =>
            !blank(
              fields[key]
            )
        )
        .length,

      total:
        keys.length
    };
  }


  validate():
    string |
    null {

    const fields =
      this.items
        ?.fields;


    if (
      !fields
    ) {
      return null;
    }


    const blank =
      (
        value:
          unknown
      ) =>
        value === null ||
        value === undefined ||
        value === '' ||
        value === '--';


    if (
      this.showWho &&
      (
        blank(
          fields[
            'reportingPerson'
          ]
        ) ||
        blank(
          fields[
            'relationship'
          ]
        )
      )
    ) {

      return translate(
        REQUIRED_MESSAGE,
        this.language
      );
    }


    if (
      this.showGait
    ) {

      const keys = [
        'walkingSupportCode',
        'fms5Id',
        'fms50Id',
        'fms500Id',
        'walkingChange'
      ];


      if (
        keys.some(
          key =>
            blank(
              fields[key]
            )
        )
      ) {

        return translate(
          REQUIRED_MESSAGE,
          this.language
        );
      }
    }


    return null;
  }


  save():
    void {

    if (
      !this.patientId ||
      !this.visitId ||
      !this.items ||
      !this.details ||
      !this.medications
    ) {

      this.saveComplete
        .emit();

      return;
    }


    if (
      !this.showWho &&
      this.copyWho
    ) {

      if (
        this.copyWho.person
      ) {

        this.items
          .fields[
            'reportingPerson'
          ] =
            this.copyWho.person;
      }


      if (
        this.copyWho.relationship
      ) {

        this.items
          .fields[
            'relationship'
          ] =
            this.copyWho.relationship;
      }
    }


    for (
      const key of [
        'reportingPerson',
        'relationship',
        'walkingSupportCode',
        'walkingChange'
      ]
    ) {

      if (
        this.items
          .fields[key] ===
          null ||
        this.items
          .fields[key] ===
          undefined
      ) {

        this.items
          .fields[key] =
            '';
      }
    }


    for (
      const key of [
        'followup',
        'generalConcerns'
      ]
    ) {

      if (
        this.details
          .fields[key] ===
          null ||
        this.details
          .fields[key] ===
          undefined
      ) {

        this.details
          .fields[key] =
            '';
      }
    }


    this.saving =
      true;

    this.saveError =
      '';


    const pending = [
      ...this.pendingConditions
    ];


    const addConditions =
      from(
        pending
      )
      .pipe(

        concatMap(
          code =>
            this.http
              .post<HealthConditionEntry>(
                `${this.patientBase()}/history/health-conditions`,
                {
                  conditionCode:
                    code
                }
              )
              .pipe(
                tap(
                  () =>
                    this.pendingConditions
                      .delete(
                        code
                      )
                )
              )
        ),

        toArray()
      );


    const put =
      (
        part:
          string,

        body:
          Section
      ) =>
        this.http
          .put<void>(
            `${this.base()}/visits/${this.visitId}/${part}`,
            body
          )
          .pipe(

            map(
              () =>
                null as
                  string |
                  null
            ),

            catchError(
              error => {

                console.error(
                  `Unable to save History (${part}):`,
                  error
                );


                return of(
                  part as
                    string |
                    null
                );
              }
            )
          );


    addConditions
      .pipe(

        switchMap(
          () =>
            forkJoin([
              put(
                'items',
                this.items!
              ),

              put(
                'details',
                this.details!
              ),

              put(
                'medications',
                this.medications!
              )
            ])
        )
      )
      .subscribe({

        next:
          results => {

            if (
              pending.length >
              0
            ) {

              this.reloadPatientHistory();
            }


            const failed =
              results.filter(
                (
                  part
                ):
                  part is
                    string =>
                      part !==
                      null
              );


            this.saving =
              false;


            if (
              failed.length >
              0
            ) {

              this.saveError =
                `Unable to save (${failed.join(', ')}). Please try again.`;


              this.cdr
                .markForCheck();


              this.saveFailed
                .emit(
                  this.saveError
                );


              return;
            }


            this.saved =
              true;


            this.cdr
              .markForCheck();


            this.progressChange
              .emit();


            this.saveComplete
              .emit();
          },


        error:
          error => {

            console.error(
              'Unable to add health conditions:',
              error
            );


            if (
              pending.length !==
              this.pendingConditions
                .size
            ) {

              this.reloadPatientHistory();
            }


            this.saving =
              false;


            this.saveError =
              'Unable to save the ticked conditions. Please try again.';


            this.cdr
              .markForCheck();


            this.saveFailed
              .emit(
                this.saveError
              );
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

    <div class="loading-text">
      Loading…
    </div>

  } @else if (loadError) {

    <div class="questionnaire-error">
      {{ loadError }}
    </div>

  } @else if (
    items &&
    details &&
    medications &&
    options
  ) {

    @if (saved) {

      <div class="submitted-banner">
        History saved.
      </div>

    }


    @if (
      saveError &&
      !hideActions
    ) {

      <div class="questionnaire-error">
        {{ saveError }}
      </div>

    }


    @if (showGeneral) {


      @if (showWho) {

        <section class="history-page intro-page">

          <p class="page-text">
            {{
              text(
                "If you have any questions while completing the questionnaire, please don't hesitate to ask a member of our staff for help."
              )
            }}
          </p>

          <p class="page-text">
            {{
              text(
                'These questions will be asked on each visit so we can stay up-to-date with you.'
              )
            }}
          </p>

        </section>


        <section class="history-page">

          <h3 class="page-heading">
            {{ text('Please tell us about you') }}
          </h3>


          <div class="field-row">

            <label
              class="field-label"
              for="reportingPerson"
            >
              {{ text('Your Name:') }}
            </label>

            <input
              id="reportingPerson"
              type="text"
              [ngModel]="items.fields['reportingPerson']"
              (ngModelChange)="setItem('reportingPerson', $event)"
            />

          </div>


          <div class="field-row">

            <label
              class="field-label"
              for="relationship"
            >
              {{
                text(
                  'What is your relationship to [name]?'
                )
              }}

              <span class="field-hint">
                {{ text('Choose from the list') }}
              </span>
            </label>

            <select
              id="relationship"
              [ngModel]="items.fields['relationship']"
              (ngModelChange)="setItem('relationship', $event)"
            >

              <option [ngValue]="null">
              </option>

              @for (
                choice of options.relationships;
                track choice
              ) {

                <option [ngValue]="choice">
                  {{
                    lab(
                      'relationship',
                      choice
                    )
                  }}
                </option>

              }

            </select>

          </div>

        </section>

      }


      <section class="history-page">

        <h3 class="page-heading">
          {{ text('Health History') }}
        </h3>


        @if (
          healthHistory.length > 0
        ) {

          <p class="page-text">
            {{
              text(
                "Here is what we show for [name]'s Health History:"
              )
            }}
          </p>

          <table class="entry-table">

            <thead>
              <tr>
                <th>
                  {{ text('Problem') }}
                </th>

                <th>
                  {{ text('Age') }}
                </th>
              </tr>
            </thead>

            <tbody>

              @for (
                entry of healthHistory;
                track entry.id
              ) {

                <tr>

                  <td>
                    {{
                      historyConditionName(
                        entry.conditionCode,
                        entry.conditionDescription
                      )
                    }}
                  </td>

                  <td>
                    {{
                      lab(
                        'age',
                        entry.age
                      )
                    }}
                  </td>

                </tr>

              }

            </tbody>

          </table>

        }


        <p class="page-text">

          {{
            healthHistory.length > 0
              ? text(
                  'Has [name] had any of the following problems that are not included in the chart above?'
                )
              : text(
                  'Has [name] had any of the following problems?'
                )
          }}

        </p>


        <h4 class="sub-heading">
          {{ text('Add to the Health History') }}
        </h4>


        <p class="page-text">
          {{
            text(
              'Please select the problem and age that [name] was when it happened. If you are not sure, select "Not Sure" in the Age list.'
            )
          }}
        </p>


        <div class="add-history">

          <div class="radio-column">

            <span class="field-label">
              {{ text('Select One Problem') }}
            </span>

            @for (
              option of historyConditionOptions;
              track option.code
            ) {

              <label class="choice-row">

                <input
                  type="radio"
                  name="historyCondition"
                  [value]="option.code"
                  [checked]="newHistoryConditionCode === option.code"
                  (change)="newHistoryConditionCode = option.code"
                />

                {{ option.name }}

              </label>

            }

          </div>


          <div class="age-column">

            <span class="field-label">
              {{ text('Age') }}
            </span>

            <select
              [(ngModel)]="newHistoryAge"
            >

              <option value="">
              </option>

              @for (
                age of ageOptions;
                track age
              ) {

                <option [value]="age">
                  {{
                    lab(
                      'age',
                      age
                    )
                  }}
                </option>

              }

            </select>

            <button
              type="button"
              class="add-btn"
              [disabled]="historySaving"
              (click)="addHealthHistory()"
            >
              {{ text('Add This To Record') }}
            </button>

          </div>

        </div>


        @if (historyError) {

          <div class="questionnaire-error">
            {{ historyError }}
          </div>

        }

      </section>


      <section class="history-page">

        <h3 class="page-heading">
          {{ text('Health Conditions') }}
        </h3>


        @if (
          healthConditions.length > 0
        ) {

          <p class="page-text">
            {{
              text(
                'Here is the information in [name]\'s record to date. If the list is ok, move on to the next question. If you need to delete one of them from the list, click "Delete" in that row.'
              )
            }}
          </p>


          <table class="entry-table">

            <thead>

              <tr>

                <th>
                  {{ text('Condition') }}
                </th>

                <th>
                </th>

              </tr>

            </thead>

            <tbody>

              @for (
                entry of healthConditions;
                track entry.id
              ) {

                <tr>

                  <td>
                    {{
                      conditionName(
                        entry.conditionCode,
                        entry.conditionDescription
                      )
                    }}
                  </td>

                  <td class="row-action">

                    <button
                      type="button"
                      class="link-btn"
                      (click)="askDeleteCondition(entry.id)"
                    >
                      {{ text('Delete') }}
                    </button>

                  </td>

                </tr>

              }

            </tbody>

          </table>

        }


        @if (
          confirmingCondition;
          as pending
        ) {

          <div class="confirm-box">

            <h4 class="sub-heading">
              {{ text('Confirm Condition Deletion') }}
            </h4>

            <p class="page-text">

              <strong>
                {{
                  text(
                    'Are you sure you want to remove the following entry from your list?'
                  )
                }}
              </strong>

            </p>

            <p class="page-text">
              {{
                conditionName(
                  pending.conditionCode,
                  pending.conditionDescription
                )
              }}
            </p>

            <div class="confirm-actions">

              <button
                type="button"
                class="add-btn"
                (click)="confirmDeleteCondition()"
              >
                {{
                  text(
                    'Yes - Delete this from my List'
                  )
                }}
              </button>

              <button
                type="button"
                class="link-btn"
                (click)="cancelDeleteCondition()"
              >
                {{
                  text(
                    'Do Not Delete this Item'
                  )
                }}
              </button>

            </div>

          </div>

        }


        <p class="page-text">

          {{
            healthConditions.length > 0
              ? text(
                  'Does [name] have any other conditions that are listed below?'
                )
              : text(
                  'Does [name] have any conditions that are listed below?'
                )
          }}

          {{
            text(
              'If so, please select them then move to the next question.'
            )
          }}

        </p>


        <div class="check-grid">

          @for (
            option of availableConditionOptions();
            track option.code
          ) {

            <label class="choice-row">

              <input
                type="checkbox"
                [checked]="isConditionPending(option.code)"
                (change)="togglePendingCondition(option.code)"
              />

              {{ option.name }}

            </label>

          }

        </div>


        @if (conditionError) {

          <div class="questionnaire-error">
            {{ conditionError }}
          </div>

        }

      </section>


      @if (hasSeizuresCondition) {

        <section class="history-page">

          <h3 class="page-heading">
            {{ text('Seizure Medications') }}
          </h3>


          @if (
            seizureMedRows.length > 0
          ) {

            <p class="page-text">
              {{
                text(
                  'Here is the information you\'ve given so far today. If the list is ok, move on to the next question. If you need to delete one of them from the list, click "Delete" in that row.'
                )
              }}
            </p>


            <table class="entry-table">

              <thead>

                <tr>

                  <th>
                    {{ text('Medication Name') }}
                  </th>

                  <th>
                  </th>

                </tr>

              </thead>

              <tbody>

                @for (
                  row of seizureMedRows;
                  track row['name']
                ) {

                  <tr>

                    <td>
                      {{ row['name'] }}
                    </td>

                    <td class="row-action">

                      <button
                        type="button"
                        class="link-btn"
                        (click)="
                          askRemove(
                            'seizure',
                            rowName(row),
                            rowName(row)
                          )
                        "
                      >
                        {{ text('Delete') }}
                      </button>

                    </td>

                  </tr>

                }

              </tbody>

            </table>

          }


          @if (
            pendingRemoval &&
            pendingRemoval.kind ===
              'seizure'
          ) {

            <div class="confirm-box">

              <h4 class="sub-heading">
                {{ removalText('title') }}
              </h4>

              <p class="page-text">

                <strong>
                  {{ removalText('question') }}
                </strong>

              </p>

              <p class="page-text">
                {{ pendingRemoval.label }}
              </p>

              <div class="confirm-actions">

                <button
                  type="button"
                  class="add-btn"
                  (click)="confirmRemove()"
                >
                  {{ removalText('confirm') }}
                </button>

                <button
                  type="button"
                  class="link-btn"
                  (click)="cancelRemove()"
                >
                  {{ removalText('cancel') }}
                </button>

              </div>

            </div>

          }


          @if (!showCustomSeizureMed) {

            <p class="page-text">

              {{
                seizureMedRows.length > 0
                  ? text(
                      'Does [name] currently use any other seizure medications? If so, please select them then move to the next question.'
                    )
                  : text(
                      'Does [name] currently use any seizure medications? If so, please select them then move to the next question.'
                    )
              }}

            </p>


            <div class="check-grid">

              @for (
                med of availableSeizureMeds();
                track med
              ) {

                <label class="choice-row">

                  <input
                    type="checkbox"
                    [checked]="false"
                    (change)="addSeizureMed(med)"
                  />

                  {{ med }}

                </label>

              }

            </div>


            <button
              type="button"
              class="link-btn"
              (click)="showCustomSeizureMed = true"
            >
              {{ text('Other') }}
            </button>

          } @else {

            <div class="add-row">

              <input
                type="text"
                [(ngModel)]="customSeizureMed"
              />

              <button
                type="button"
                class="add-btn"
                (click)="addCustomSeizureMed()"
              >
                {{ text('Add') }}
              </button>

              <button
                type="button"
                class="link-btn"
                (click)="showCustomSeizureMed = false"
              >
                {{ text('Cancel') }}
              </button>

            </div>

          }

        </section>

      }


      <section class="history-page">

        <h3 class="page-heading">
          {{ text('Botox History') }}
        </h3>


        @if (botox.length > 0) {

          <table class="entry-table">

            <thead>

              <tr>

                <th>
                  {{ text('Location') }}
                </th>

                <th>
                  {{ text('Side') }}
                </th>

                <th>
                  {{ text('Date') }}
                </th>

                <th>
                  {{ text('Facility') }}
                </th>

                <th>
                  {{ text('Physician') }}
                </th>

              </tr>

            </thead>

            <tbody>

              @for (
                entry of botox;
                track entry.id
              ) {

                <tr>

                  <td>
                    {{
                      lab(
                        'bodyLoc',
                        entry.bodyLocation
                      )
                    }}
                  </td>

                  <td>
                    {{ entry.side }}
                  </td>

                  <td>
                    {{ entry.date }}
                  </td>

                  <td>
                    {{ entry.facility }}
                  </td>

                  <td>
                    {{ entry.physician }}
                  </td>

                </tr>

              }

            </tbody>

          </table>

        }


        @if (!showBotoxForm) {

          <button
            type="button"
            class="add-btn"
            (click)="showBotoxForm = true"
          >
            {{ text('Add Botox History') }}
          </button>

        } @else {

          <div class="botox-form">

            <div class="add-row botox-add-row">

              <select
                [(ngModel)]="newBotox.bodyLocation"
              >

                <option value="">
                  {{ text('Location') }}
                </option>

                @for (
                  location of bodyLocationOptions;
                  track location
                ) {

                  <option [value]="location">
                    {{
                      lab(
                        'bodyLoc',
                        location
                      )
                    }}
                  </option>

                }

              </select>


              <select
                [(ngModel)]="newBotox.side"
              >

                <option value="">
                  {{ text('Side') }}
                </option>

                @for (
                  side of botoxSideOptions;
                  track side.code
                ) {

                  <option [value]="side.code">
                    {{ side.name }}
                  </option>

                }

              </select>


              <input
                type="date"
                [(ngModel)]="newBotox.date"
              />

              <input
                type="text"
                placeholder="Facility"
                [(ngModel)]="newBotox.facility"
              />

              <input
                type="text"
                placeholder="Physician"
                [(ngModel)]="newBotox.physician"
              />

            </div>


            <div class="confirm-actions">

              <button
                type="button"
                class="add-btn"
                [disabled]="botoxSaving"
                (click)="addBotox()"
              >
                {{ text('Add') }}
              </button>

              <button
                type="button"
                class="link-btn"
                (click)="showBotoxForm = false"
              >
                {{ text('Cancel') }}
              </button>

            </div>


            @if (botoxError) {

              <div class="questionnaire-error">
                {{ botoxError }}
              </div>

            }

          </div>

        }

      </section>

    }


    @if (showGait) {

      <section class="history-page">

        <h3 class="page-heading">
          {{ text('Walking') }}
        </h3>


        <div class="question-block">

          <span class="question-label">
            {{
              text(
                'Does [name] need support while walking?'
              )
            }}
          </span>

          <div class="radio-column">

            @for (
              choice of options.walkingSupport;
              track choice.value
            ) {

              <label class="choice-row">

                <input
                  type="radio"
                  name="walkingSupportCode"
                  [ngValue]="choice.value"
                  [checked]="
                    items.fields['walkingSupportCode'] === choice.value
                  "
                  (change)="
                    setItem(
                      'walkingSupportCode',
                      choice.value
                    )
                  "
                />

                {{
                  choice.label ??
                  choice.value
                }}

              </label>

            }

          </div>

        </div>


        @for (
          question of fmsQuestions;
          track question.key
        ) {

          <div class="question-block">

            <span class="question-label">
              {{ text(question.text) }}
            </span>

            <div class="radio-column">

              @for (
                choice of options.fms;
                track choice.value
              ) {

                <label class="choice-row">

                  <input
                    type="radio"
                    [name]="question.key"
                    [ngValue]="choice.value"
                    [checked]="
                      items.fields[question.key] === choice.value
                    "
                    (change)="
                      setItem(
                        question.key,
                        choice.value
                      )
                    "
                  />

                  {{
                    choice.label ??
                    choice.value
                  }}

                </label>

              }

            </div>

          </div>

        }


        <div class="question-block">

          <span class="question-label">
            {{
              text(
                "How has [name]'s walking changed since the last visit?"
              )
            }}
          </span>

          <div class="radio-column">

            @for (
              choice of options.walkingChanges;
              track choice.value
            ) {

              <label class="choice-row">

                <input
                  type="radio"
                  name="walkingChange"
                  [ngValue]="choice.value"
                  [checked]="
                    items.fields['walkingChange'] === choice.value
                  "
                  (change)="
                    setItem(
                      'walkingChange',
                      choice.value
                    )
                  "
                />

                {{ walkingChangeLabel(choice) }}

              </label>

            }

          </div>

        </div>

      </section>

    }


    @if (showConcerns) {

      <section class="history-page">

        <h3 class="page-heading">
          {{ text('Current Devices / Braces') }}
        </h3>


        @if (
          deviceRows.length > 0
        ) {

          <ul class="chip-list">

            @for (
              row of deviceRows;
              track row['name']
            ) {

              <li class="chip">

                <span>

                  {{ row['name'] }}

                  @if (row['side']) {
                    -
                    {{ row['side'] }}
                  }

                </span>

                <button
                  type="button"
                  class="chip-remove"
                  (click)="
                    askRemove(
                      'device',
                      row['name'],
                      String(row['name'])
                    )
                  "
                >
                  ×
                </button>

              </li>

            }

          </ul>

        }


        @if (
          pendingRemoval &&
          pendingRemoval.kind ===
            'device'
        ) {

          <div class="confirm-box">

            <h4 class="sub-heading">
              {{ removalText('title') }}
            </h4>

            <p class="page-text">

              <strong>
                {{ removalText('question') }}
              </strong>

            </p>

            <p class="page-text">
              {{ pendingRemoval.label }}
            </p>

            <div class="confirm-actions">

              <button
                type="button"
                class="add-btn"
                (click)="confirmRemove()"
              >
                {{ removalText('confirm') }}
              </button>

              <button
                type="button"
                class="link-btn"
                (click)="cancelRemove()"
              >
                {{ removalText('cancel') }}
              </button>

            </div>

          </div>

        }


        <div class="checklist">

          @for (
            device of availableDevices();
            track device
          ) {

            <div class="checklist-row">

              <label class="checkbox-option">

                <input
                  type="checkbox"
                  [checked]="false"
                  (change)="addDevice(device)"
                />

                {{ lab('device', device) }}

              </label>


              @if (
                deviceHasSide(device)
              ) {

                <select
                  class="side-select"
                  [ngModel]="pendingSide(device)"
                  (ngModelChange)="setPendingSide(device, $event)"
                >

                  @for (
                    side of deviceSideOptions;
                    track side.value
                  ) {

                    <option [value]="side.value">
                      {{ side.label }}
                    </option>

                  }

                </select>

              }

            </div>

          }

        </div>


        @if (!showCustomDevice) {

          <button
            type="button"
            class="link-btn"
            (click)="showCustomDevice = true"
          >
            {{ text('Other') }}
          </button>

        } @else {

          <div class="add-row">

            <input
              type="text"
              [(ngModel)]="customDevice"
            />

            <button
              type="button"
              class="add-btn"
              (click)="addCustomDevice()"
            >
              {{ text('Add') }}
            </button>

            <button
              type="button"
              class="link-btn"
              (click)="showCustomDevice = false"
            >
              {{ text('Cancel') }}
            </button>

          </div>

        }

      </section>


      <section class="history-page">

        <h3 class="page-heading">
          {{ text('Pain') }}
        </h3>


        <div class="pain-list">

          @for (
            bodyPart of options.pain;
            track bodyPart
          ) {

            <div class="pain-row">

              <span class="pain-part">
                {{ lab('pain', bodyPart) }}
              </span>


              <div class="pain-sides">

                @for (
                  side of painSideChoices(bodyPart);
                  track side.value
                ) {

                  <label class="pain-side-option">

                    <input
                      type="radio"
                      [name]="'pain-' + bodyPart"
                      [value]="side.value"
                      [checked]="
                        painSide(bodyPart) === side.value
                      "
                      (change)="
                        setPainSide(
                          bodyPart,
                          side.value
                        )
                      "
                    />

                    {{ side.label }}

                  </label>

                }

              </div>

            </div>

          }

        </div>

      </section>


      <section class="history-page">

        <h3 class="page-heading">
          {{ text('Gait Concerns') }}
        </h3>


        @if (
          gaitConcernRows.length > 0
        ) {

          <ul class="chip-list">

            @for (
              row of gaitConcernRows;
              track row['code']
            ) {

              <li class="chip">

                <span>
                  {{
                    gaitConcernLabel(
                      row['code']
                    )
                  }}
                </span>

                <button
                  type="button"
                  class="chip-remove"
                  (click)="
                    askRemove(
                      'concern',
                      row['code'],
                      gaitConcernLabel(
                        row['code']
                      )
                    )
                  "
                >
                  ×
                </button>

              </li>

            }

          </ul>

        }


        @if (
          pendingRemoval &&
          pendingRemoval.kind ===
            'concern'
        ) {

          <div class="confirm-box">

            <h4 class="sub-heading">
              {{ removalText('title') }}
            </h4>

            <p class="page-text">

              <strong>
                {{ removalText('question') }}
              </strong>

            </p>

            <p class="page-text">
              {{ pendingRemoval.label }}
            </p>

            <div class="confirm-actions">

              <button
                type="button"
                class="add-btn"
                (click)="confirmRemove()"
              >
                {{ removalText('confirm') }}
              </button>

              <button
                type="button"
                class="link-btn"
                (click)="cancelRemove()"
              >
                {{ removalText('cancel') }}
              </button>

            </div>

          </div>

        }


        <div class="check-grid">

          @for (
            concern of availableGaitConcerns();
            track concern.value
          ) {

            <label class="choice-row">

              <input
                type="checkbox"
                [checked]="false"
                (change)="addGaitConcern(concern.value)"
              />

              {{
                concern.label ??
                concern.value
              }}

            </label>

          }

        </div>

      </section>


      <section class="history-page">

        <h3 class="page-heading">
          {{ text('Therapy Frequency') }}
        </h3>


        @for (
          group of therapyGroups;
          track group.title
        ) {

          <h4 class="sub-heading">
            {{ group.title }}
          </h4>

          <div class="therapy-grid">

            @for (
              field of group.fields;
              track field.key
            ) {

              <div class="therapy-item">

                <label [for]="field.key">
                  {{ field.label }}
                </label>

                <select
                  [id]="field.key"
                  [(ngModel)]="details.fields[field.key]"
                  (ngModelChange)="progressChange.emit()"
                >

                  @for (
                    choice of frequencyChoices;
                    track choice.value
                  ) {

                    <option [value]="choice.value">
                      {{ choice.label }}
                    </option>

                  }

                </select>

              </div>

            }

          </div>

        }

      </section>


      <section class="history-page">

        <h3 class="page-heading">
          {{ text('Follow Up') }}
        </h3>


        <div class="field-row">

          <label
            class="field-label"
            for="followup"
          >
            {{ text('Date of Followup with MD') }}
          </label>

          <input
            id="followup"
            type="date"
            class="followup-input"
            [(ngModel)]="details.fields['followup']"
            (ngModelChange)="progressChange.emit()"
          />

        </div>


        <div class="field-row">

          <label
            class="field-label"
            for="generalConcerns"
          >
            {{ text('General Concerns') }}
          </label>

          <textarea
            id="generalConcerns"
            rows="5"
            [(ngModel)]="details.fields['generalConcerns']"
            (ngModelChange)="progressChange.emit()"
          >
          </textarea>

        </div>

      </section>

    }


    @if (!hideActions) {

      <div class="history-actions">

        <button
          type="button"
          class="submit-btn"
          [disabled]="saving"
          (click)="save()"
        >
          {{
            saving
              ? 'Saving...'
              : (
                  saved
                    ? 'Save Again'
                    : 'Save'
                )
          }}
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

.question-block {
  display: flex;
  flex-direction: column;

  gap: 10px;

  padding: 16px 18px;

  background: #ffffff;

  border: 1px solid #e2e8f0;
  border-radius: 10px;

  transition:
    border-color 0.15s ease,
    box-shadow 0.15s ease;
}

.question-block:hover {
  border-color: #cbd5e1;

  box-shadow:
    0 1px 6px
    rgba(15, 23, 42, 0.05);
}

.question-block:focus-within {
  border-color: #269c96;

  box-shadow:
    0 0 0 3px
    rgba(38, 156, 150, 0.12);
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

  box-shadow:
    0 0 0 3px
    rgba(38, 156, 150, 0.12);
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
  font-weight: 500;

  color: #374151;
}

.therapy-item select {
  max-width: 240px;

  margin: 0;
}

.checklist {
  display: grid;

  grid-template-columns:
    repeat(
      2,
      minmax(0, 1fr)
    );

  gap: 4px 16px;
}

.checklist-row {
  display: flex;
  align-items: center;
  justify-content: space-between;

  gap: 12px;

  padding: 9px 12px;

  border-radius: 8px;

  transition:
    background-color 0.12s ease;
}

.checklist-row:has(
  .checkbox-option input:checked
) {
  background: #eef6f4;
}

.checkbox-option {
  display: flex;
  align-items: center;

  gap: 10px;

  color: #374151;

  font-size: 13px;

  cursor: pointer;
}

.checklist > .checkbox-option {
  padding: 9px 12px;

  border-radius: 8px;

  transition:
    background-color 0.12s ease;
}

.checklist > .checkbox-option:hover {
  background: #f8fafc;
}

.checklist > .checkbox-option:has(
  input:checked
) {
  background: #eef6f4;

  color: #1f6b5e;

  font-weight: 500;
}

.checkbox-option input[type="checkbox"] {
  width: 16px;
  height: 16px;

  flex-shrink: 0;

  accent-color: #269c96;

  cursor: pointer;
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
  color: #374151;

  font-size: 13px;
  font-weight: 500;
}

.pain-sides {
  display: flex;

  gap: 16px;
}

.pain-side-option {
  display: flex;
  align-items: center;

  gap: 6px;

  color: #374151;

  font-size: 13px;

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

  transition:
    background-color 0.12s ease,
    border-color 0.12s ease;
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

  transition:
    background-color 0.12s ease;
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

  color: #374151;

  font-size: 13px;
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

  transition:
    background-color 0.12s ease,
    color 0.12s ease;
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

  box-shadow:
    0 1px 3px
    rgba(31, 132, 127, 0.25);

  transition:
    background-color 0.15s ease,
    box-shadow 0.15s ease;
}

.submit-btn:hover:not(:disabled) {
  background: #1f847f;

  box-shadow:
    0 2px 6px
    rgba(31, 132, 127, 0.3);
}

.submit-btn:disabled {
  background: #b7d4d2;

  border-color: #b7d4d2;

  box-shadow: none;

  cursor: not-allowed;
}

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
  width: 100%;
  max-width: none;

  resize: vertical;
}

.history-page input:focus,
.history-page select:focus,
.history-page textarea:focus {
  outline: none;

  border-color: #269c96;

  box-shadow:
    0 0 0 3px
    rgba(38, 156, 150, 0.12);
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
  width: 1%;

  text-align: right;

  white-space: nowrap;
}

.link-btn {
  padding: 0;

  background: none;

  border: none;

  color: #1f847f;

  font-family: inherit;
  font-size: 13px;

  text-align: left;
  text-decoration: underline;

  cursor: pointer;
}

.link-btn:hover {
  color: #14524f;
}

.check-grid {
  display: grid;

  grid-template-columns:
    repeat(
      2,
      minmax(0, 1fr)
    );

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

  color: #374151;

  font-size: 13px;

  cursor: pointer;
}

.choice-row:hover {
  background: #f8fafc;
}

.choice-row input {
  width: 16px;
  height: 16px;

  flex-shrink: 0;

  accent-color: #269c96;

  cursor: pointer;
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
  flex-wrap: wrap;

  gap: 12px;
}

.add-history {
  display: grid;

  grid-template-columns:
    minmax(0, 1fr)
    minmax(0, 220px);

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
    grid-template-columns:
      1fr;
  }

  .pain-row {
    align-items: flex-start;
    flex-direction: column;
  }

  .pain-sides {
    flex-wrap: wrap;
  }

  .therapy-item {
    align-items: stretch;
    flex-direction: column;
  }

  .therapy-item select {
    width: 100%;
    max-width: none;
  }
}









sports-questionnaire.ts


import { CommonModule } from '@angular/common';
import {
  ChangeDetectorRef,
  Component,
  EventEmitter,
  Input,
  OnChanges,
  Output,
  SimpleChanges
} from '@angular/core';
import { FormsModule } from '@angular/forms';
import { HttpClient } from '@angular/common/http';
import { QuestionnaireProgressValue } from '../questionnaire-progress.model';

export interface SportsAnswers {
  reportingPerson: string;
  physicalTherapy: string;
  competitiveSports: string;
  yearsInSport: string;
  daysPerWeek: string;
  orthotics: string;
  painLocations: string[];
  mileTimeKnown: 'unknown' | 'known' | '';
  mileTimeType: 'Unknown' | 'Time';
  mileTime: string;
  goals: string;
}

@Component({
  selector: 'app-sports-questionnaire',
  standalone: true,
  imports: [
    CommonModule,
    FormsModule
  ],
  templateUrl:
    './sports-questionnaire.html',
  styleUrl:
    './sports-questionnaire.css'
})
export class SportsQuestionnaire
implements OnChanges {

  @Input()
  patientId:
    number |
    null = null;

  @Input()
  visitId:
    number |
    null = null;

  @Input()
  hideActions =
    false;

  @Output()
  saveComplete =
    new EventEmitter<void>();

  @Output()
  saveFailed =
    new EventEmitter<string>();

  @Output()
  progressChange =
    new EventEmitter<void>();


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


  readonly daysPerWeekOptions = [
    'none',
    '1',
    '2',
    '3',
    '4',
    '5',
    '6',
    '7'
  ];


  readonly orthoticsOptions = [
    'None',
    'Over the counter orthotics',
    'Custom fitted orthotics'
  ];


  readonly painLocationOptions = [
    'None',
    'Back',
    'R Hip',
    'L Hip',
    'R Knee',
    'L Knee',
    'R Ankle',
    'L Ankle',
    'R Foot',
    'L Foot',
    'R Thigh',
    'L Thigh',
    'R Lower leg',
    'L Lower leg'
  ];


  answers:
    SportsAnswers =
      this.createEmptyAnswers();


  saving =
    false;

  saveError =
    '';

  saved =
    false;


  constructor(
    private http:
      HttpClient,

    private cdr:
      ChangeDetectorRef
  ) {}


  ngOnChanges(
    changes:
      SimpleChanges
  ): void {

    if (
      !(
        changes['patientId'] ||
        changes['visitId']
      )
    ) {
      return;
    }


    this.answers =
      this.createEmptyAnswers();

    this.saved =
      false;

    this.saveError =
      '';

    this.preload();
  }


  private preload():
    void {

    if (
      !this.patientId ||
      !this.visitId
    ) {
      return;
    }


    const visitId =
      this.visitId;


    this.http
      .get<
        Partial<
          Record<
            keyof SportsAnswers,
            string |
            string[] |
            null
          >
        >
      >(
        `/api/patients/${this.patientId}/sports-questionnaire/${visitId}`
      )
      .subscribe({

        next:
          saved => {

            if (
              visitId !==
                this.visitId ||
              !saved
            ) {
              return;
            }


            const text =
              (
                value:
                  string |
                  string[] |
                  null |
                  undefined
              ) =>
                typeof value ===
                'string'
                  ? value
                  : '';


            const known =
              saved.mileTimeKnown;


            this.answers = {

              reportingPerson:
                text(
                  saved.reportingPerson
                ),

              physicalTherapy:
                text(
                  saved.physicalTherapy
                ),

              competitiveSports:
                text(
                  saved.competitiveSports
                ),

              yearsInSport:
                text(
                  saved.yearsInSport
                ),

              daysPerWeek:
                text(
                  saved.daysPerWeek
                ),

              orthotics:
                text(
                  saved.orthotics
                ),

              painLocations:
                Array.isArray(
                  saved.painLocations
                )
                  ? [
                      ...saved.painLocations
                    ]
                  : [],

              mileTimeKnown:
                known === 'unknown' ||
                known === 'known'
                  ? known
                  : 'unknown',

              mileTimeType:
                known === 'known'
                  ? 'Time'
                  : 'Unknown',

              mileTime:
                text(
                  saved.mileTime
                ),

              goals:
                text(
                  saved.goals
                )
            };


            this.cdr
              .markForCheck();


            this.progressChange
              .emit();
          },


        error:
          () => {}
      });
  }


  private createEmptyAnswers():
    SportsAnswers {

    return {

      reportingPerson:
        '',

      physicalTherapy:
        '',

      competitiveSports:
        '',

      yearsInSport:
        '',

      daysPerWeek:
        '',

      orthotics:
        '',

      painLocations:
        [],

      mileTimeKnown:
        'unknown',

      mileTimeType:
        'Unknown',

      mileTime:
        '',

      goals:
        ''
    };
  }


  painLocationLabel(
    location:
      string
  ): string {

    if (
      location.startsWith(
        'R '
      )
    ) {

      return (
        `Right ` +
        `${location.substring(2)}`
      );
    }


    if (
      location.startsWith(
        'L '
      )
    ) {

      return (
        `Left ` +
        `${location.substring(2)}`
      );
    }


    return location;
  }


  isPainSelected(
    location:
      string
  ): boolean {

    return (
      this.answers
        .painLocations
        .includes(
          location
        )
    );
  }


  isPainDisabled(
    location:
      string
  ): boolean {

    return (
      location !== 'None' &&
      this.isPainSelected(
        'None'
      )
    );
  }


  togglePainLocation(
    location:
      string
  ): void {

    const selected =
      this.isPainSelected(
        location
      );


    if (
      location ===
      'None'
    ) {

      if (selected) {

        this.answers
          .painLocations =
            [];

      } else {

        this.answers
          .painLocations = [
            'None'
          ];
      }


      this.progressChange
        .emit();

      return;
    }


    if (
      this.isPainSelected(
        'None'
      )
    ) {
      return;
    }


    if (selected) {

      this.answers
        .painLocations =
          this.answers
            .painLocations
            .filter(
              item =>
                item !==
                location
            );

    } else {

      this.answers
        .painLocations = [
          ...this.answers
            .painLocations,

          location
        ];
    }


    this.progressChange
      .emit();
  }


  isMileTimeEnabled():
    boolean {

    return (
      this.answers
        .mileTimeType ===
      'Time'
    );
  }


  setMileTimeType(
    type:
      'Unknown' |
      'Time'
  ): void {

    this.answers
      .mileTimeType =
        type;


    if (
      type ===
      'Unknown'
    ) {

      this.answers
        .mileTimeKnown =
          'unknown';

      this.answers
        .mileTime =
          '';

    } else {

      this.answers
        .mileTimeKnown =
          'known';
    }


    this.progressChange
      .emit();
  }


  getProgress():
    QuestionnaireProgressValue {

    const textAnswered =
      (
        value:
          string
      ) =>
        value.trim() !==
          '' &&
        value.trim() !==
          'Tap to Enter Text';


    const a =
      this.answers;


    const answered = [

      textAnswered(
        a.reportingPerson
      ),

      a.physicalTherapy !==
        '',

      textAnswered(
        a.competitiveSports
      ),

      a.yearsInSport !==
        '',

      a.daysPerWeek !==
        '',

      a.orthotics !==
        '',

      a.painLocations
        .length >
        0,

      a.mileTimeKnown !==
        '',

      textAnswered(
        a.goals
      )

    ]
    .filter(
      Boolean
    )
    .length;


    return {
      answered,
      total: 9
    };
  }


  validate():
    string |
    null {

    const text =
      (
        value:
          string
      ) =>
        value.trim() !==
          '' &&
        value.trim() !==
          'Tap to Enter Text';


    const a =
      this.answers;


    const complete =

      text(
        a.reportingPerson
      ) &&

      a.physicalTherapy !==
        '' &&

      text(
        a.competitiveSports
      ) &&

      a.yearsInSport !==
        '' &&

      a.daysPerWeek !==
        '' &&

      a.orthotics !==
        '' &&

      a.painLocations
        .length >
        0 &&

      a.mileTimeKnown !==
        '' &&

      text(
        a.goals
      );


    return (
      complete
        ? null
        : 'You must answer above before submitting.'
    );
  }


  save():
    void {

    if (
      !this.patientId ||
      !this.visitId
    ) {

      this.saveComplete
        .emit();

      return;
    }


    this.saving =
      true;

    this.saveError =
      '';


    this.http
      .post(
        `/api/patients/${this.patientId}/sports-questionnaire`,
        {

          visitId:
            this.visitId,

          ...this.answers
        }
      )
      .subscribe({

        next:
          () => {

            this.saving =
              false;

            this.saved =
              true;


            this.cdr
              .markForCheck();


            this.progressChange
              .emit();


            this.saveComplete
              .emit();
          },


        error:
          error => {

            console.error(
              'Unable to save Sports questionnaire:',
              error
            );


            this.saving =
              false;


            this.saveError =
              'Unable to save. Please try again.';


            this.cdr
              .markForCheck();


            this.saveFailed
              .emit(
                this.saveError
              );
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


  @if (
    saveError &&
    !hideActions
  ) {

    <div class="questionnaire-error">
      {{ saveError }}
    </div>

  }


  <div class="sports-body">


    <div class="question-block">

      <label
        class="question-label"
        for="reportingPerson"
      >
        Please give the name of the person answering these questions.
      </label>

      <input
        id="reportingPerson"
        type="text"
        [(ngModel)]="answers.reportingPerson"
        (ngModelChange)="progressChange.emit()"
      />

    </div>


    <div class="question-block">

      <span class="question-label">
        Do you currently have physical therapy?
      </span>

      <div class="option-list">

        @for (
          option of physicalTherapyOptions;
          track option
        ) {

          <label class="radio-option">

            <input
              type="radio"
              name="physicalTherapy"
              [value]="option"
              [(ngModel)]="answers.physicalTherapy"
              (ngModelChange)="progressChange.emit()"
            />

            {{ option }}

          </label>

        }

      </div>

    </div>


    <div class="question-block">

      <label
        class="question-label"
        for="competitiveSports"
      >
        In the past year which competitive sports have you participated in?
      </label>

      <textarea
        id="competitiveSports"
        rows="2"
        [(ngModel)]="answers.competitiveSports"
        (ngModelChange)="progressChange.emit()"
      >
      </textarea>

    </div>


    <div class="question-block">

      <span class="question-label">
        How many years have you participated in your primary competitive sport?
      </span>

      <div class="option-list">

        @for (
          option of yearsInSportOptions;
          track option
        ) {

          <label class="radio-option">

            <input
              type="radio"
              name="yearsInSport"
              [value]="option"
              [(ngModel)]="answers.yearsInSport"
              (ngModelChange)="progressChange.emit()"
            />

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

        @for (
          option of daysPerWeekOptions;
          track option
        ) {

          <label class="radio-option">

            <input
              type="radio"
              name="daysPerWeek"
              [value]="option"
              [(ngModel)]="answers.daysPerWeek"
              (ngModelChange)="progressChange.emit()"
            />

            {{ option }}

          </label>

        }

      </div>

    </div>


    <div class="question-block">

      <span class="question-label">
        Do you currently wear orthotics?
      </span>

      <div class="option-list">

        @for (
          option of orthoticsOptions;
          track option
        ) {

          <label class="radio-option">

            <input
              type="radio"
              name="orthotics"
              [value]="option"
              [(ngModel)]="answers.orthotics"
              (ngModelChange)="progressChange.emit()"
            />

            {{ option }}

          </label>

        }

      </div>

    </div>


    <div class="question-block">

      <span class="question-label">
        Do you have current pain?
        (None must be unchecked if you wish to select anything else)
      </span>


      <div class="option-list option-list-grid">

        @for (
          option of painLocationOptions;
          track option
        ) {

          <label
            class="radio-option"
            [class.disabled-choice]="isPainDisabled(option)"
          >

            <input
              type="checkbox"
              [checked]="isPainSelected(option)"
              [disabled]="isPainDisabled(option)"
              (change)="togglePainLocation(option)"
            />

            {{ painLocationLabel(option) }}

          </label>

        }

      </div>

    </div>


    <div class="question-block">

      <span class="question-label">
        What is your best mile time?
      </span>


      <label class="radio-option">

        <input
          type="radio"
          name="mileTimeType"
          value="Unknown"
          [checked]="answers.mileTimeType === 'Unknown'"
          (change)="setMileTimeType('Unknown')"
        />

        Unknown

      </label>


      <div class="mile-time-row">

        <label class="radio-option mile-time-option">

          <input
            type="radio"
            name="mileTimeType"
            value="Time"
            [checked]="answers.mileTimeType === 'Time'"
            (change)="setMileTimeType('Time')"
          />

          Time:

          <input
            type="text"
            class="inline-input"
            [(ngModel)]="answers.mileTime"
            [disabled]="!isMileTimeEnabled()"
            (ngModelChange)="progressChange.emit()"
          />

        </label>

      </div>

    </div>


    <div class="question-block">

      <label
        class="question-label"
        for="goals"
      >
        What are your current goals?
      </label>

      <textarea
        id="goals"
        rows="2"
        [(ngModel)]="answers.goals"
        (ngModelChange)="progressChange.emit()"
      >
      </textarea>

    </div>

  </div>


  @if (!hideActions) {

    <div class="sports-actions">

      <button
        type="button"
        class="submit-btn"
        [disabled]="saving"
        (click)="save()"
      >
        {{
          saving
            ? 'Saving...'
            : (
                saved
                  ? 'Save Again'
                  : 'Save'
              )
        }}
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

  transition:
    border-color 0.15s ease,
    box-shadow 0.15s ease;
}

.question-block:hover {
  border-color: #cbd5e1;

  box-shadow:
    0 1px 6px
    rgba(15, 23, 42, 0.05);
}

.question-block:focus-within {
  border-color: #269c96;

  box-shadow:
    0 0 0 3px
    rgba(38, 156, 150, 0.12);
}

.question-label {
  color: #1e293b;

  font-size: 14px;
  font-weight: 600;
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

.question-block textarea {
  resize: vertical;
}

.question-block input[type="text"]:focus,
.question-block textarea:focus {
  outline: none;

  border-color: #269c96;

  box-shadow:
    0 0 0 3px
    rgba(38, 156, 150, 0.12);
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
  display: grid;

  grid-template-columns:
    repeat(
      2,
      minmax(0, 1fr)
    );

  gap: 6px 12px;
}

.radio-option {
  display: flex;
  align-items: center;

  gap: 10px;

  padding: 9px 12px;

  border-radius: 8px;

  color: #374151;

  font-size: 13px;

  cursor: pointer;

  transition:
    background-color 0.12s ease;
}

.radio-option:hover {
  background: #f8fafc;
}

.radio-option:has(
  input:checked
) {
  background: #eef6f4;

  color: #1f6b5e;

  font-weight: 500;
}

.radio-option input[type="radio"],
.radio-option input[type="checkbox"] {
  width: 16px;
  height: 16px;

  flex-shrink: 0;

  accent-color: #269c96;

  cursor: pointer;
}

.option-list-inline
.radio-option {
  padding: 8px 14px;

  border: 1px solid #e2e8f0;
}

.option-list-inline
.radio-option:has(
  input:checked
) {
  border-color: #269c96;
}

.mile-time-row {
  display: flex;
  align-items: center;
}

.mile-time-option {
  gap: 10px;
}

.inline-input {
  width: 110px !important;

  height: 32px;

  padding: 0 10px !important;

  box-sizing: border-box;

  border: 1px solid #d7dde2;
  border-radius: 6px;

  font-family: inherit;
  font-size: 13px;
}

.inline-input:disabled {
  background: #f3f4f6;

  color: #9ca3af;

  cursor: not-allowed;
}

.disabled-choice {
  color: #9ca3af;

  background: #f8fafc;

  cursor: not-allowed;
}

.disabled-choice:hover {
  background: #f8fafc;
}

.disabled-choice input {
  cursor: not-allowed !important;
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

  box-shadow:
    0 1px 3px
    rgba(31, 132, 127, 0.25);

  transition:
    background-color 0.15s ease,
    box-shadow 0.15s ease;
}

.submit-btn:hover:not(:disabled) {
  background: #1f847f;

  box-shadow:
    0 2px 6px
    rgba(31, 132, 127, 0.3);
}

.submit-btn:disabled {
  background: #b7d4d2;

  border-color: #b7d4d2;

  box-shadow: none;

  cursor: not-allowed;
}

@media (max-width: 700px) {

  .option-list-grid {
    grid-template-columns:
      1fr;
  }

  .option-list-inline {
    flex-direction: column;
  }

  .option-list-inline
  .radio-option {
    width: 100%;

    box-sizing: border-box;
  }
}







hip-questionnaire.ts




import { CommonModule } from '@angular/common';
import {
  ChangeDetectorRef,
  Component,
  EventEmitter,
  Input,
  OnChanges,
  Output
} from '@angular/core';
import { FormsModule } from '@angular/forms';
import { HttpClient } from '@angular/common/http';
import {
  QuestionnaireProgressValue
} from '../questionnaire-progress.model';

interface Scale5Item {
  key: string;
  text: string;
}

interface HarrisItem {
  key: string;
  label: string;

  options: {
    points: number;
    label: string;
  }[];
}

@Component({
  selector: 'app-hip-questionnaire',
  standalone: true,
  imports: [
    CommonModule,
    FormsModule
  ],
  templateUrl: './hip-questionnaire.html',
  styleUrl: './hip-questionnaire.css'
})
export class HipQuestionnaire
implements OnChanges {

  @Input()
  patientId:
    number |
    null = null;

  @Input()
  visitId:
    number |
    null = null;

  @Input()
  hideActions =
    false;

  @Output()
  saveComplete =
    new EventEmitter<void>();

  @Output()
  saveFailed =
    new EventEmitter<string>();

  @Output()
  progressChange =
    new EventEmitter<void>();


  readonly womacScale = [
    'None',
    'Mild',
    'Moderate',
    'Severe',
    'Extreme'
  ];


  readonly womacPain:
    Scale5Item[] = [

    {
      key: 'womac_01',
      text:
        'Walking on a flat surface'
    },

    {
      key: 'womac_02',
      text:
        'Going up and down stairs'
    },

    {
      key: 'womac_03',
      text:
        'At night while in bed, pain disturbs your sleep'
    },

    {
      key: 'womac_04',
      text:
        'Sitting or lying'
    },

    {
      key: 'womac_05',
      text:
        'Standing upright'
    }
  ];


  readonly womacStiffness:
    Scale5Item[] = [

    {
      key: 'womac_06',
      text:
        'How severe is your stiffness after first awakening in the morning?'
    },

    {
      key: 'womac_07',
      text:
        'How severe is your stiffness after sitting, lying, or resting in the day?'
    }
  ];


  readonly womacFunction:
    Scale5Item[] = [

    {
      key: 'womac_08',
      text:
        'Going down stairs'
    },

    {
      key: 'womac_09',
      text:
        'Going up stairs'
    },

    {
      key: 'womac_10',
      text:
        'Rising from sitting'
    },

    {
      key: 'womac_11',
      text:
        'Standing'
    },

    {
      key: 'womac_12',
      text:
        'Bending to the floor'
    },

    {
      key: 'womac_13',
      text:
        'Walking on flat surfaces'
    },

    {
      key: 'womac_14',
      text:
        'Getting in and out of a car, or on or off a bus'
    },

    {
      key: 'womac_15',
      text:
        'Going shopping'
    },

    {
      key: 'womac_16',
      text:
        'Putting on your socks or stockings'
    },

    {
      key: 'womac_17',
      text:
        'Rising from the bed'
    },

    {
      key: 'womac_18',
      text:
        'Taking off your socks or stockings'
    },

    {
      key: 'womac_19',
      text:
        'Lying in bed'
    },

    {
      key: 'womac_20',
      text:
        'Getting in or out of the bath'
    },

    {
      key: 'womac_21',
      text:
        'Sitting'
    },

    {
      key: 'womac_22',
      text:
        'Getting on or off the toilet'
    },

    {
      key: 'womac_23',
      text:
        'Performance heavy domestic duties'
    },

    {
      key: 'womac_24',
      text:
        'Performing light domestic duties'
    }
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


  readonly harrisItems:
    HarrisItem[] = [

    {
      key: 'harris_01',
      label: 'Pain',

      options: [

        {
          points: 44,
          label: 'None/ignores'
        },

        {
          points: 40,
          label:
            'Slight, occasional, no compromise in activity'
        },

        {
          points: 30,
          label:
            'Mild, no effect on ordinary activity, pain after activity, uses aspirin'
        },

        {
          points: 20,
          label:
            'Moderate, tolerable, makes concessions, occasional codeine'
        },

        {
          points: 10,
          label:
            'Marked, serious limitations'
        },

        {
          points: 0,
          label:
            'Totally disabled'
        }
      ]
    },

    {
      key: 'harris_02',
      label: 'Limp',

      options: [

        {
          points: 11,
          label: 'None'
        },

        {
          points: 8,
          label: 'Slight'
        },

        {
          points: 5,
          label: 'Moderate'
        },

        {
          points: 0,
          label:
            'Severe / Unable to walk'
        }
      ]
    },

    {
      key: 'harris_03',
      label: 'Support',

      options: [

        {
          points: 11,
          label: 'None'
        },

        {
          points: 7,
          label:
            'Cane, long walks'
        },

        {
          points: 5,
          label:
            'Cane, full time'
        },

        {
          points: 4,
          label: 'Crutch'
        },

        {
          points: 2,
          label: '2 canes'
        },

        {
          points: 1,
          label: '2 crutches'
        },

        {
          points: 0,
          label:
            'Unable to walk'
        }
      ]
    },

    {
      key: 'harris_04',
      label: 'Distance Walked',

      options: [

        {
          points: 11,
          label: 'Unlimited'
        },

        {
          points: 8,
          label: '6 blocks'
        },

        {
          points: 5,
          label: '2-3 blocks'
        },

        {
          points: 2,
          label:
            'Indoors only'
        },

        {
          points: 0,
          label:
            'Bed and chair'
        }
      ]
    },

    {
      key: 'harris_05',
      label: 'Stairs',

      options: [

        {
          points: 4,
          label: 'Normally'
        },

        {
          points: 2,
          label:
            'Normally with banister'
        },

        {
          points: 1,
          label: 'Any method'
        },

        {
          points: 0,
          label: 'Not able'
        }
      ]
    },

    {
      key: 'harris_06',
      label: 'Sock/Shoes',

      options: [

        {
          points: 4,
          label: 'With ease'
        },

        {
          points: 2,
          label:
            'With difficulty'
        },

        {
          points: 0,
          label: 'Unable'
        }
      ]
    },

    {
      key: 'harris_07',
      label: 'Sitting',

      options: [

        {
          points: 5,
          label:
            'Any chair, 1 hour'
        },

        {
          points: 3,
          label:
            'High chair, 1/2 hour'
        },

        {
          points: 0,
          label:
            'Unable to sit, 1/2 hour, any chair'
        }
      ]
    },

    {
      key: 'harris_08',
      label:
        'Public Transportation',

      options: [

        {
          points: 1,
          label:
            'Able to enter public transportation'
        },

        {
          points: 0,
          label:
            'Unable to use public transportation'
        }
      ]
    }
  ];


  womac:
    Record<
      string,
      number |
      null
    > = {};


  ucla:
    number |
    null = null;


  harris:
    Record<
      string,
      number |
      null
    > = {};


  saving =
    false;

  saveError =
    '';

  saved =
    false;


  constructor(
    private http:
      HttpClient,

    private cdr:
      ChangeDetectorRef
  ) {

    this.resetAnswers();
  }


  ngOnChanges():
    void {

    this.resetAnswers();

    this.saved =
      false;

    this.saveError =
      '';

    this.preload();
  }


  private preload():
    void {

    if (
      !this.patientId ||
      !this.visitId
    ) {
      return;
    }


    const visitId =
      this.visitId;


    this.http
      .get<{

        womac?:
          Record<
            string,
            number |
            null
          >;

        harris?:
          Record<
            string,
            number |
            null
          >;

        ucla?:
          number |
          null;

      }>(
        `/api/patients/${this.patientId}/hip-questionnaire/${visitId}`
      )
      .subscribe({

        next:
          saved => {

            if (
              visitId !==
              this.visitId
            ) {
              return;
            }


            for (
              const [
                key,
                value
              ] of
                Object.entries(
                  saved.womac ??
                  {}
                )
            ) {

              if (
                key in this.womac &&
                typeof value ===
                  'number'
              ) {

                this.womac[
                  key
                ] =
                  value;
              }
            }


            for (
              const [
                key,
                value
              ] of
                Object.entries(
                  saved.harris ??
                  {}
                )
            ) {

              if (
                key in this.harris &&
                typeof value ===
                  'number'
              ) {

                this.harris[
                  key
                ] =
                  value;
              }
            }


            if (
              typeof saved.ucla ===
              'number'
            ) {

              this.ucla =
                saved.ucla;
            }


            this.cdr
              .markForCheck();


            this.progressChange
              .emit();
          },


        error:
          () => {}
      });
  }


  private resetAnswers():
    void {

    this.womac =
      {};


    for (
      const item of [
        ...this.womacPain,
        ...this.womacStiffness,
        ...this.womacFunction
      ]
    ) {

      this.womac[
        item.key
      ] =
        null;
    }


    this.ucla =
      null;


    this.harris =
      {};


    for (
      const item of
        this.harrisItems
    ) {

      this.harris[
        item.key
      ] =
        null;
    }
  }


  answerChanged():
    void {

    this.saved =
      false;


    this.progressChange
      .emit();
  }


  getProgress():
    QuestionnaireProgressValue {

    const womacItems = [
      ...this.womacPain,
      ...this.womacStiffness,
      ...this.womacFunction
    ];


    const womacAnswered =
      womacItems
        .filter(
          item =>
            this.womac[
              item.key
            ] !==
              null &&
            this.womac[
              item.key
            ] !==
              undefined
        )
        .length;


    const uclaAnswered =
      this.ucla !==
        null &&
      this.ucla !==
        undefined
          ? 1
          : 0;


    const harrisAnswered =
      this.harrisItems
        .filter(
          item =>
            this.harris[
              item.key
            ] !==
              null &&
            this.harris[
              item.key
            ] !==
              undefined
        )
        .length;


    return {

      answered:
        womacAnswered +
        uclaAnswered +
        harrisAnswered,

      total:
        womacItems.length +
        1 +
        this.harrisItems.length
    };
  }


  validate():
    string |
    null {

    const progress =
      this.getProgress();


    if (
      progress.answered <
      progress.total
    ) {

      return (
        'Please answer all Hip questionnaire questions before moving to the next questionnaire.'
      );
    }


    return null;
  }


  save():
    void {

    if (
      !this.patientId ||
      !this.visitId
    ) {

      this.saveComplete
        .emit();

      return;
    }


    this.saving =
      true;

    this.saveError =
      '';


    this.http
      .post(
        `/api/patients/${this.patientId}/hip-questionnaire`,
        {

          visitId:
            this.visitId,

          womac:
            this.womac,

          ucla:
            this.ucla,

          harris:
            this.harris
        }
      )
      .subscribe({

        next:
          () => {

            this.saving =
              false;

            this.saved =
              true;


            this.cdr
              .markForCheck();


            this.progressChange
              .emit();


            this.saveComplete
              .emit();
          },


        error:
          error => {

            console.error(
              'Unable to save Hip questionnaire:',
              error
            );


            this.saving =
              false;


            this.saveError =
              'Unable to save. Please try again.';


            this.cdr
              .markForCheck();


            this.saveFailed
              .emit(
                this.saveError
              );
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


  @if (
    saveError &&
    !hideActions
  ) {

    <div class="questionnaire-error">
      {{ saveError }}
    </div>

  }


  <section class="hip-section">

    <h4>
      HOW MUCH PAIN HAVE YOU HAD IN YOUR HIP/KNEE IN THE LAST 2 DAYS?
    </h4>


    @for (
      item of womacPain;
      track item.key
    ) {

      <div class="scale-row">

        <span class="scale-text">
          {{ item.text }}
        </span>


        <div class="scale-options">

          @for (
            label of womacScale;
            track label;
            let optionIndex = $index
          ) {

            <label class="scale-option">

              <input
                type="radio"
                [name]="item.key"
                [value]="optionIndex"
                [(ngModel)]="womac[item.key]"
                (ngModelChange)="answerChanged()"
              />

              {{ label }}

            </label>

          }

        </div>

      </div>

    }

  </section>


  <section class="hip-section">

    <h4>
      HOW MUCH STIFFNESS/DIFFICULTY MOVING YOUR HIP/KNEE HAVE YOU HAD IN THE LAST 2 DAYS?
    </h4>


    @for (
      item of womacStiffness;
      track item.key
    ) {

      <div class="scale-row">

        <span class="scale-text">
          {{ item.text }}
        </span>


        <div class="scale-options">

          @for (
            label of womacScale;
            track label;
            let optionIndex = $index
          ) {

            <label class="scale-option">

              <input
                type="radio"
                [name]="item.key"
                [value]="optionIndex"
                [(ngModel)]="womac[item.key]"
                (ngModelChange)="answerChanged()"
              />

              {{ label }}

            </label>

          }

        </div>

      </div>

    }

  </section>


  <section class="hip-section">

    <h4>
      HOW MUCH DIFFICULTY HAVE YOU HAD WITH YOUR DAILY PHYSICAL ACTIVITIES OVER THE LAST 2 DAYS?
    </h4>


    @for (
      item of womacFunction;
      track item.key
    ) {

      <div class="scale-row">

        <span class="scale-text">
          {{ item.text }}
        </span>


        <div class="scale-options">

          @for (
            label of womacScale;
            track label;
            let optionIndex = $index
          ) {

            <label class="scale-option">

              <input
                type="radio"
                [name]="item.key"
                [value]="optionIndex"
                [(ngModel)]="womac[item.key]"
                (ngModelChange)="answerChanged()"
              />

              {{ label }}

            </label>

          }

        </div>

      </div>

    }

  </section>


  <section class="hip-section">

    <h4>
      Please check one box that best describes current activity level.
    </h4>


    <div class="option-list">

      @for (
        label of uclaOptions;
        track label;
        let optionIndex = $index
      ) {

        <label class="radio-option">

          <input
            type="radio"
            name="ucla"
            [value]="optionIndex + 1"
            [(ngModel)]="ucla"
            (ngModelChange)="answerChanged()"
          />

          {{ label }}

        </label>

      }

    </div>

  </section>


  <section class="hip-section harris-section">

    <h4>
      HARRIS HIP SCORE
    </h4>


    @for (
      item of harrisItems;
      track item.key
    ) {

      <div class="harris-item">

        <span class="harris-label">
          {{ item.label }}
        </span>


        <div class="option-list">

          @for (
            option of item.options;
            track option.points
          ) {

            <label class="radio-option">

              <input
                type="radio"
                [name]="item.key"
                [value]="option.points"
                [(ngModel)]="harris[item.key]"
                (ngModelChange)="answerChanged()"
              />

              {{ option.label }}

            </label>

          }

        </div>

      </div>

    }

  </section>


  @if (!hideActions) {

    <div class="hip-actions">

      <button
        type="button"
        class="submit-btn"
        [disabled]="saving"
        (click)="save()"
      >
        {{
          saving
            ? 'Saving...'
            : (
                saved
                  ? 'Save Again'
                  : 'Save'
              )
        }}
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

  gap: 22px;
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
  line-height: 1.45;
}

.scale-row {
  display: flex;
  flex-direction: column;

  gap: 10px;

  padding: 14px 18px;

  background: #ffffff;

  border: 1px solid #e2e8f0;
  border-radius: 10px;

  transition:
    border-color 0.15s ease,
    box-shadow 0.15s ease;
}

.scale-row:hover {
  border-color: #cbd5e1;

  box-shadow:
    0 1px 6px
    rgba(15, 23, 42, 0.05);
}

.scale-row:focus-within {
  border-color: #269c96;

  box-shadow:
    0 0 0 3px
    rgba(38, 156, 150, 0.12);
}

.scale-text {
  color: #1e293b;

  font-size: 14px;
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

  color: #374151;

  font-size: 12px;

  cursor: pointer;

  transition:
    background-color 0.12s ease,
    border-color 0.12s ease;
}

.scale-option:hover {
  background: #f8fafc;
}

.scale-option:has(
  input:checked
) {
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

  color: #374151;

  font-size: 13px;
  line-height: 1.4;

  cursor: pointer;

  transition:
    background-color 0.12s ease;
}

.radio-option:hover {
  background: #f8fafc;
}

.radio-option:has(
  input:checked
) {
  background: #eef6f4;

  color: #1f6b5e;

  font-weight: 500;
}

.radio-option input {
  width: 16px;
  height: 16px;

  margin-top: 1px;

  flex-shrink: 0;

  accent-color: #269c96;

  cursor: pointer;
}

.harris-section {
  padding-bottom: 2px;
}

.harris-item {
  display: flex;
  flex-direction: column;

  gap: 8px;

  padding: 14px 18px;

  background: #ffffff;

  border: 1px solid #e2e8f0;
  border-radius: 10px;

  transition:
    border-color 0.15s ease,
    box-shadow 0.15s ease;
}

.harris-item:hover {
  border-color: #cbd5e1;

  box-shadow:
    0 1px 6px
    rgba(15, 23, 42, 0.05);
}

.harris-item:focus-within {
  border-color: #269c96;

  box-shadow:
    0 0 0 3px
    rgba(38, 156, 150, 0.12);
}

.harris-label {
  color: #1e293b;

  font-size: 14px;
  font-weight: 600;
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

  box-shadow:
    0 1px 3px
    rgba(31, 132, 127, 0.25);

  transition:
    background-color 0.15s ease,
    box-shadow 0.15s ease;
}

.submit-btn:hover:not(:disabled) {
  background: #1f847f;

  box-shadow:
    0 2px 6px
    rgba(31, 132, 127, 0.3);
}

.submit-btn:disabled {
  background: #b7d4d2;

  border-color: #b7d4d2;

  box-shadow: none;

  cursor: not-allowed;
}

@media (max-width: 700px) {

  .scale-options {
    flex-direction: column;
  }

  .scale-option {
    width: 100%;

    box-sizing: border-box;

    border-radius: 8px;
  }

  .hip-section h4 {
    font-size: 14px;
  }
}








podci-questionnaire.ts


import { CommonModule } from '@angular/common';
import {
  ChangeDetectorRef,
  Component,
  EventEmitter,
  Input,
  OnChanges,
  Output,
  SimpleChanges
} from '@angular/core';
import { FormsModule } from '@angular/forms';
import { HttpClient } from '@angular/common/http';

import {
  PODCI_BANK,
  PodciBankVariant,
  PodciBlock,
  PodciPage
} from './podci-bank';

import {
  QuestionnaireProgressValue
} from '../questionnaire-progress.model';

export type PodciVariant =
  PodciBankVariant;

interface FollowUp {
  followUpPage: number;
  key: string;
  showFor: number[];
}

const FOLLOW_UPS:
  Record<number, FollowUp> = {

    13: {
      followUpPage: 14,
      key: 'q2_061',
      showFor: [4]
    },

    15: {
      followUpPage: 16,
      key: 'q2_069',
      showFor: [4]
    },

    17: {
      followUpPage: 18,
      key: 'q2_077',
      showFor: [4]
    },

    19: {
      followUpPage: 20,
      key: 'q2_085',
      showFor: [2, 3]
    },

    21: {
      followUpPage: 22,
      key: 'q2_091',
      showFor: [2, 3]
    }
  };

const FOLLOW_UP_PAGES =
  new Set(
    Object
      .values(FOLLOW_UPS)
      .map(
        followUp =>
          followUp.followUpPage
      )
  );

type Answer =
  number |
  boolean |
  string |
  null;

export interface RenderedPodciPage {
  n: number;
  blocks: PodciBlock[];
}

@Component({
  selector:
    'app-podci-questionnaire',

  standalone:
    true,

  imports: [
    CommonModule,
    FormsModule
  ],

  templateUrl:
    './podci-questionnaire.html',

  styleUrl:
    './podci-questionnaire.css'
})
export class PodciQuestionnaire
implements OnChanges {

  @Input()
  patientId:
    number |
    null = null;

  @Input()
  visitId:
    number |
    null = null;

  @Input()
  firstName = '';

  @Input()
  variant:
    PodciVariant =
      'CH';

  @Input()
  language:
    'en' |
    'sp' =
      'en';

  @Input()
  hideActions =
    false;

  @Output()
  saveComplete =
    new EventEmitter<void>();

  @Output()
  saveFailed =
    new EventEmitter<string>();

  @Output()
  progressChange =
    new EventEmitter<void>();


  answers:
    Record<
      string,
      Answer
    > = {};


  saving =
    false;

  saveError =
    '';

  saved =
    false;


  constructor(
    private http:
      HttpClient,

    private cdr:
      ChangeDetectorRef
  ) {}


  ngOnChanges(
    changes:
      SimpleChanges
  ): void {

    if (
      !(
        changes['patientId'] ||
        changes['visitId'] ||
        changes['variant']
      )
    ) {
      return;
    }


    this.answers =
      {};

    this.saved =
      false;

    this.saveError =
      '';

    this.preload();
  }


  private preload():
    void {

    if (
      !this.patientId ||
      !this.visitId
    ) {
      return;
    }


    const visitId =
      this.visitId;

    const variant =
      this.variant;


    this.http
      .get<
        Record<
          string,
          Answer
        >
      >(
        `/api/patients/${this.patientId}/podci-questionnaire/${visitId}`,
        {
          params: {
            variant:
              variant.toLowerCase()
          }
        }
      )
      .subscribe({

        next:
          saved => {

            if (
              visitId !==
                this.visitId ||
              variant !==
                this.variant
            ) {
              return;
            }


            const loaded:
              Record<
                string,
                Answer
              > = {};


            for (
              const [
                key,
                value
              ] of
                Object.entries(
                  saved ??
                  {}
                )
            ) {

              if (
                value !==
                  null &&
                value !==
                  undefined
              ) {

                loaded[key] =
                  value;
              }
            }


            this.answers = {
              ...loaded,
              ...this.answers
            };


            this.cdr
              .markForCheck();


            this.progressChange
              .emit();
          },


        error:
          () => {}
      });
  }


  private blocksFor(
    page:
      PodciPage
  ): PodciBlock[] {

    return (
      this.language ===
        'sp' &&
      page.blocks.sp

        ? page.blocks.sp

        : page.blocks.en
    );
  }


  get pages():
    RenderedPodciPage[] {

    const out:
      RenderedPodciPage[] =
        [];


    for (
      const page of
        PODCI_BANK[
          this.variant
        ]
    ) {

      if (
        FOLLOW_UP_PAGES
          .has(page.n) &&
        !this.followUpShown(
          page.n
        )
      ) {
        continue;
      }


      const blocks =
        this.blocksFor(
          page
        );


      out.push({

        n:
          page.n,

        blocks:
          page.n > 1 &&
          blocks[0]?.t ===
            'text'

            ? blocks.slice(1)

            : blocks
      });
    }


    return out;
  }


  private followUpShown(
    pageNumber:
      number
  ): boolean {

    const parent =
      Object
        .entries(
          FOLLOW_UPS
        )
        .find(
          (
            [
              ,
              followUp
            ]
          ) =>
            followUp
              .followUpPage ===
            pageNumber
        );


    if (!parent) {
      return true;
    }


    const value =
      this.answers[
        parent[1].key
      ];


    return (
      typeof value ===
        'number' &&
      parent[1]
        .showFor
        .includes(
          value
        )
    );
  }


  text(
    template:
      string
  ): string {

    return (
      template.replace(
        /\[name\]/g,

        this.firstName ||
          (
            this.language ===
              'sp'

              ? 'su niño(a)'

              : 'your child'
          )
      )
    );
  }


  selected(
    name:
      string,

    value:
      number
  ): boolean {

    return (
      this.answers[
        name
      ] === value
    );
  }


  setRadio(
    name:
      string,

    value:
      number
  ): void {

    this.answers[
      name
    ] =
      value;


    this.saved =
      false;


    this.progressChange
      .emit();
  }


  setGrid(
    name:
      string,

    value:
      number
  ): void {

    this.answers[
      name
    ] =
      value;


    if (
      name.endsWith(
        'a'
      ) &&
      value ===
        2
    ) {

      const base =
        name.slice(
          0,
          -1
        );


      this.answers[
        base +
        'b'
      ] =
        null;


      this.answers[
        base +
        'c'
      ] =
        null;
    }


    this.saved =
      false;


    this.progressChange
      .emit();
  }


  gridDisabled(
    name:
      string
  ): boolean {

    return (
      /[bc]$/
        .test(name) &&
      this.answers[
        name.slice(
          0,
          -1
        ) +
        'a'
      ] ===
        2
    );
  }


  isChecked(
    name:
      string
  ): boolean {

    return (
      this.answers[
        name
      ] ===
        true
    );
  }


  setChecked(
    name:
      string,

    checked:
      boolean
  ): void {

    this.answers[
      name
    ] =
      checked;


    this.saved =
      false;


    this.progressChange
      .emit();
  }


  comment(
    name:
      string
  ): string {

    return (
      this.answers[
        name
      ] as
        string
    ) ?? '';
  }


  setComment(
    name:
      string,

    value:
      string
  ): void {

    this.answers[
      name
    ] =
      value;


    this.saved =
      false;


    this.progressChange
      .emit();
  }


  getProgress():
    QuestionnaireProgressValue {

    let answered =
      0;

    let total =
      0;


    for (
      const page of
        this.pages
    ) {

      const bankPage =
        PODCI_BANK[
          this.variant
        ]
        .find(
          bankPage =>
            bankPage.n ===
            page.n
        );


      for (
        const name of
          bankPage?.required ??
          []
      ) {

        if (
          this.gridDisabled(
            name
          )
        ) {
          continue;
        }


        total++;


        const value =
          this.answers[
            name
          ];


        if (
          value !==
            null &&
          value !==
            undefined &&
          value !==
            ''
        ) {

          answered++;
        }
      }
    }


    return {
      answered,
      total
    };
  }


  private buildAnswers():
    Record<
      string,
      Answer
    > {

    const out:
      Record<
        string,
        Answer
      > = {};


    for (
      const page of
        this.pages
    ) {

      for (
        const block of
          page.blocks
      ) {

        switch (
          block.t
        ) {

          case 'radios':

            if (
              this.answers[
                block.name
              ] != null
            ) {

              out[
                block.name
              ] =
                this.answers[
                  block.name
                ];
            }

            break;


          case 'scale':

            for (
              const row of
                block.rows
            ) {

              if (
                this.answers[
                  row.name
                ] != null
              ) {

                out[
                  row.name
                ] =
                  this.answers[
                    row.name
                  ];
              }
            }

            break;


          case 'grid':

            for (
              const row of
                block.rows
            ) {

              for (
                const cell of
                  row.cells
              ) {

                if (
                  this.gridDisabled(
                    cell.name
                  )
                ) {

                  out[
                    cell.name
                  ] =
                    null;

                } else if (
                  this.answers[
                    cell.name
                  ] != null
                ) {

                  out[
                    cell.name
                  ] =
                    this.answers[
                      cell.name
                    ];
                }
              }
            }

            break;


          case 'checks':

            for (
              const item of
                block.items
            ) {

              out[
                item.name
              ] =
                this.answers[
                  item.name
                ] ===
                  true;
            }

            break;


          case 'textarea':

            out[
              block.name
            ] =
              (
                this.answers[
                  block.name
                ] as
                  string
              ) ??
              '';

            break;
        }
      }
    }


    return out;
  }


  validate():
    string |
    null {

    for (
      const page of
        this.pages
    ) {

      const bankPage =
        PODCI_BANK[
          this.variant
        ]
        .find(
          bankPage =>
            bankPage.n ===
            page.n
        );


      for (
        const name of
          bankPage?.required ??
          []
      ) {

        if (
          this.gridDisabled(
            name
          )
        ) {
          continue;
        }


        const value =
          this.answers[
            name
          ];


        if (
          value ===
            null ||
          value ===
            undefined
        ) {

          return (
            'Please respond to all questions before continuing.'
          );
        }
      }
    }


    return null;
  }


  save():
    void {

    if (
      !this.patientId ||
      !this.visitId
    ) {

      this.saveComplete
        .emit();

      return;
    }


    this.saving =
      true;

    this.saveError =
      '';


    this.http
      .post(
        `/api/patients/${this.patientId}/podci-questionnaire`,
        {

          visitId:
            this.visitId,

          variant:
            this.variant
              .toLowerCase(),

          answers:
            this.buildAnswers()
        }
      )
      .subscribe({

        next:
          () => {

            this.saving =
              false;

            this.saved =
              true;


            this.cdr
              .markForCheck();


            this.progressChange
              .emit();


            this.saveComplete
              .emit();
          },


        error:
          error => {

            console.error(
              'Unable to save PODCI questionnaire:',
              error
            );


            this.saving =
              false;


            this.saveError =
              'Unable to save. Please try again.';


            this.cdr
              .markForCheck();


            this.saveFailed
              .emit(
                this.saveError
              );
          }
      });
  }
}








<div class="podci-form">

  @if (saved) {

    <div class="submitted-banner">
      PODCI questionnaire saved.
    </div>

  }


  @for (
    page of pages;
    track page.n
  ) {

    <section class="podci-page">

      @for (
        block of page.blocks;
        track $index
      ) {


        @if (
          block.t ===
          'text'
        ) {

          <p class="stem">
            {{ text(block.text) }}
          </p>


        } @else if (
          block.t ===
          'radios'
        ) {

          <div class="option-list">

            @for (
              option of block.options;
              track option.value
            ) {

              <label class="radio-option">

                <input
                  type="radio"
                  [name]="block.name"
                  [value]="option.value"
                  [checked]="
                    selected(
                      block.name,
                      option.value
                    )
                  "
                  (change)="
                    setRadio(
                      block.name,
                      option.value
                    )
                  "
                />

                {{ text(option.label) }}

              </label>

            }

          </div>


        } @else if (
          block.t ===
          'scale'
        ) {

          <div class="table-wrap">

            <table class="scale-table">

              <thead>

                <tr>

                  <th>
                  </th>

                  @for (
                    column of block.columns;
                    track $index
                  ) {

                    <th>
                      {{ text(column) }}
                    </th>

                  }

                </tr>

              </thead>


              <tbody>

                @for (
                  row of block.rows;
                  track row.name
                ) {

                  <tr>

                    <td class="row-label">
                      {{ text(row.text) }}
                    </td>


                    @for (
                      column of block.columns;
                      track $index;
                      let i = $index
                    ) {

                      <td class="radio-cell">

                        <input
                          type="radio"
                          [name]="row.name"
                          [value]="i + 1"
                          [attr.aria-label]="column"
                          [checked]="
                            selected(
                              row.name,
                              i + 1
                            )
                          "
                          (change)="
                            setRadio(
                              row.name,
                              i + 1
                            )
                          "
                        />

                      </td>

                    }

                  </tr>

                }

              </tbody>

            </table>

          </div>


        } @else if (
          block.t ===
          'grid'
        ) {

          <div class="table-wrap">

            <table class="scale-table">

              <thead>

                <tr>

                  <th>
                  </th>

                  @for (
                    column of block.columns;
                    track $index
                  ) {

                    <th>
                      {{ text(column) }}
                    </th>

                  }

                </tr>

              </thead>


              <tbody>

                @for (
                  row of block.rows;
                  track row.label
                ) {

                  <tr>

                    <td class="row-label">
                      {{ text(row.label) }}
                    </td>


                    @for (
                      cell of row.cells;
                      track cell.name
                    ) {

                      <td class="radio-cell">

                        <div class="yes-no">

                          @for (
                            option of cell.options;
                            track option.value
                          ) {

                            <label
                              class="yes-no-option"
                              [class.disabled]="
                                gridDisabled(
                                  cell.name
                                )
                              "
                            >

                              <input
                                type="radio"
                                [name]="cell.name"
                                [value]="option.value"
                                [checked]="
                                  selected(
                                    cell.name,
                                    option.value
                                  )
                                "
                                [disabled]="
                                  gridDisabled(
                                    cell.name
                                  )
                                "
                                (change)="
                                  setGrid(
                                    cell.name,
                                    option.value
                                  )
                                "
                              />

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


        } @else if (
          block.t ===
          'checks'
        ) {

          @if (
            block.items[0]
              ?.rowLabel
          ) {

            <div class="limiter-list">

              @for (
                item of block.items;
                track item.name
              ) {

                <label class="limiter-row">

                  <span class="limiter-text">
                    {{
                      text(
                        item.rowLabel
                      )
                    }}
                  </span>

                  <span class="limiter-check">

                    <input
                      type="checkbox"
                      [checked]="
                        isChecked(
                          item.name
                        )
                      "
                      (change)="
                        setChecked(
                          item.name,
                          $any(
                            $event.target
                          ).checked
                        )
                      "
                    />

                    {{ text(item.label) }}

                  </span>

                </label>

              }

            </div>


          } @else {

            <div class="region-grid">

              @for (
                item of block.items;
                track item.name
              ) {

                <label class="region-option">

                  <input
                    type="checkbox"
                    [checked]="
                      isChecked(
                        item.name
                      )
                    "
                    (change)="
                      setChecked(
                        item.name,
                        $any(
                          $event.target
                        ).checked
                      )
                    "
                  />

                  {{ text(item.label) }}

                </label>

              }

            </div>

          }


        } @else if (
          block.t ===
          'textarea'
        ) {

          <textarea
            class="comment-box"
            rows="3"
            [ngModel]="
              comment(
                block.name
              )
            "
            (ngModelChange)="
              setComment(
                block.name,
                $event
              )
            "
          >
          </textarea>

        }

      }

    </section>

  }


  @if (
    saveError &&
    !hideActions
  ) {

    <div class="questionnaire-error">
      {{ saveError }}
    </div>

  }


  @if (!hideActions) {

    <div class="podci-actions">

      <button
        type="button"
        class="submit-btn"
        [disabled]="saving"
        (click)="save()"
      >
        {{
          saving
            ? 'Saving...'
            : (
                saved
                  ? 'Save Again'
                  : 'Save'
              )
        }}
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

  transition:
    background-color 0.12s ease;
}

.radio-option:hover {
  background: #f8fafc;
}

.radio-option:has(
  input:checked
) {
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

  grid-template-columns:
    repeat(
      3,
      minmax(0, 1fr)
    );

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

  box-shadow:
    0 0 0 3px
    rgba(38, 156, 150, 0.12);
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

  box-shadow:
    0 1px 3px
    rgba(31, 132, 127, 0.25);

  transition:
    background-color 0.15s ease,
    box-shadow 0.15s ease;
}

.submit-btn:hover:not(:disabled) {
  background: #1f847f;

  box-shadow:
    0 2px 6px
    rgba(31, 132, 127, 0.3);
}

.submit-btn:disabled {
  background: #b7d4d2;

  border-color: #b7d4d2;

  box-shadow: none;

  cursor: not-allowed;
}

@media (max-width: 700px) {

  .region-grid {
    grid-template-columns:
      1fr;
  }

  .limiter-row {
    align-items: flex-start;
    flex-direction: column;

    gap: 6px;
  }
}










ue-history-questionnaire.ts




import { CommonModule } from '@angular/common';
import {
  ChangeDetectorRef,
  Component,
  EventEmitter,
  Input,
  OnChanges,
  Output
} from '@angular/core';
import { HttpClient } from '@angular/common/http';

import {
  QuestionnaireProgressValue
} from '../questionnaire-progress.model';

interface SingleChoice {
  key:
    'handlesObjects' |
    'dresses' |
    'bathes' |
    'toilets';

  question:
    string;

  options:
    string[];
}

const DEPENDENCE = [
  'Yes, independently',
  'Yes but needs some assistance',
  'No, is dependent'
];

@Component({
  selector:
    'app-ue-history-questionnaire',

  standalone:
    true,

  imports: [
    CommonModule
  ],

  templateUrl:
    './ue-history-questionnaire.html',

  styleUrl:
    './ue-history-questionnaire.css'
})
export class UeHistoryQuestionnaire
implements OnChanges {

  @Input()
  patientId:
    number |
    null = null;

  @Input()
  visitId:
    number |
    null = null;

  @Input()
  hideActions =
    false;

  @Output()
  saveComplete =
    new EventEmitter<void>();

  @Output()
  saveFailed =
    new EventEmitter<string>();

  @Output()
  progressChange =
    new EventEmitter<void>();


  readonly handsQuestion:
    SingleChoice = {

      key:
        'handlesObjects',

      question:
        'My child handles objects with both hands',

      options: [
        'Yes',
        'Yes but with difficulty',
        'No RIGHT hand only',
        'No LEFT hand only'
      ]
    };


  readonly selfCareQuestions:
    SingleChoice[] = [

      {
        key:
          'dresses',

        question:
          'My child is able to dress him/herself',

        options:
          DEPENDENCE
      },

      {
        key:
          'bathes',

        question:
          'My child is able to bathe him/herself',

        options:
          DEPENDENCE
      },

      {
        key:
          'toilets',

        question:
          'My child is able to use the toilet him/herself',

        options:
          DEPENDENCE
      }
    ];


  readonly concernsQuestion =
    'Concerns with arm /hand';


  readonly concernOptions = [
    'Elbow flexed',
    'Wrist flexed',
    'Thumb in palm',
    'Hand fisted',
    'Weak grasp',
    'Palm down',
    'Neglects arm',
    'Hard to get in a sleeve'
  ];


  answers:
    Record<
      string,
      string |
      null
    > = {};


  concerns =
    new Set<string>();


  saving =
    false;

  saveError =
    '';

  saved =
    false;


  constructor(
    private http:
      HttpClient,

    private cdr:
      ChangeDetectorRef
  ) {}


  ngOnChanges():
    void {

    this.answers =
      {};

    this.concerns =
      new Set<string>();

    this.saved =
      false;

    this.saveError =
      '';

    this.preload();
  }


  private preload():
    void {

    if (
      !this.patientId ||
      !this.visitId
    ) {
      return;
    }


    const visitId =
      this.visitId;


    this.http
      .get<
        Record<
          string,
          string |
          string[] |
          null
        >
      >(
        `/api/patients/${this.patientId}/ue-history-questionnaire/${visitId}`
      )
      .subscribe({

        next:
          saved => {

            if (
              visitId !==
              this.visitId
            ) {
              return;
            }


            for (
              const key of [
                'handlesObjects',
                'dresses',
                'bathes',
                'toilets'
              ]
            ) {

              const value =
                saved?.[key];


              if (
                typeof value ===
                  'string' &&
                value
              ) {

                this.answers[
                  key
                ] =
                  value;
              }
            }


            const chosen =
              saved?.[
                'armHandConcerns'
              ];


            if (
              Array.isArray(
                chosen
              )
            ) {

              this.concerns =
                new Set(
                  chosen
                );
            }


            this.cdr
              .markForCheck();


            this.progressChange
              .emit();
          },


        error:
          () => {}
      });
  }


  choose(
    key:
      string,

    value:
      string
  ): void {

    this.answers[
      key
    ] =
      value;


    this.saved =
      false;


    this.progressChange
      .emit();
  }


  toggleConcern(
    option:
      string
  ): void {

    if (
      this.concerns
        .has(
          option
        )
    ) {

      this.concerns
        .delete(
          option
        );

    } else {

      this.concerns
        .add(
          option
        );
    }


    this.saved =
      false;


    this.progressChange
      .emit();
  }


  getProgress():
    QuestionnaireProgressValue {

    const requiredQuestions = [
      this.handsQuestion,
      ...this.selfCareQuestions
    ];


    const answered =
      requiredQuestions
        .filter(
          question =>
            !!this.answers[
              question.key
            ]
        )
        .length;


    return {
      answered,
      total:
        requiredQuestions.length
    };
  }


  validate():
    string |
    null {

    const questions = [
      this.handsQuestion,
      ...this.selfCareQuestions
    ];


    return (
      questions.some(
        question =>
          !this.answers[
            question.key
          ]
      )

        ? 'You must answer above before submitting.'

        : null
    );
  }


  save():
    void {

    if (
      !this.patientId ||
      !this.visitId
    ) {

      this.saveComplete
        .emit();

      return;
    }


    this.saving =
      true;

    this.saveError =
      '';


    this.http
      .post(
        `/api/patients/${this.patientId}/ue-history-questionnaire`,
        {

          visitId:
            this.visitId,

          handlesObjects:
            this.answers[
              'handlesObjects'
            ] ??
            null,

          dresses:
            this.answers[
              'dresses'
            ] ??
            null,

          bathes:
            this.answers[
              'bathes'
            ] ??
            null,

          toilets:
            this.answers[
              'toilets'
            ] ??
            null,

          armHandConcerns:
            [
              ...this.concerns
            ]
        }
      )
      .subscribe({

        next:
          () => {

            this.saving =
              false;

            this.saved =
              true;


            this.cdr
              .markForCheck();


            this.progressChange
              .emit();


            this.saveComplete
              .emit();
          },


        error:
          error => {

            console.error(
              'Unable to save UE History:',
              error
            );


            this.saving =
              false;


            this.saveError =
              'Unable to save. Please try again.';


            this.cdr
              .markForCheck();


            this.saveFailed
              .emit(
                this.saveError
              );
          }
      });
  }
}










<div class="ue-form">

  @if (saved) {

    <div class="submitted-banner">
      UE History saved.
    </div>

  }


  @if (
    saveError &&
    !hideActions
  ) {

    <div class="questionnaire-error">
      {{ saveError }}
    </div>

  }


  <section class="ue-page">

    <h3 class="page-heading">
      {{ handsQuestion.question }}
    </h3>


    <div class="radio-column">

      @for (
        option of handsQuestion.options;
        track option
      ) {

        <label class="choice-row">

          <input
            type="radio"
            [name]="handsQuestion.key"
            [checked]="
              answers[
                handsQuestion.key
              ] === option
            "
            (change)="
              choose(
                handsQuestion.key,
                option
              )
            "
          />

          {{ option }}

        </label>

      }

    </div>

  </section>


  <section class="ue-page">

    @for (
      question of selfCareQuestions;
      track question.key
    ) {

      <div class="question-block">

        <h3 class="page-heading">
          {{ question.question }}
        </h3>


        <div class="radio-column">

          @for (
            option of question.options;
            track option
          ) {

            <label class="choice-row">

              <input
                type="radio"
                [name]="question.key"
                [checked]="
                  answers[
                    question.key
                  ] === option
                "
                (change)="
                  choose(
                    question.key,
                    option
                  )
                "
              />

              {{ option }}

            </label>

          }

        </div>

      </div>

    }

  </section>


  <section class="ue-page">

    <h3 class="page-heading">
      {{ concernsQuestion }}
    </h3>


    <div class="concerns-grid">

      @for (
        option of concernOptions;
        track option
      ) {

        <label class="choice-row">

          <input
            type="checkbox"
            [checked]="
              concerns.has(
                option
              )
            "
            (change)="
              toggleConcern(
                option
              )
            "
          />

          {{ option }}

        </label>

      }

    </div>

  </section>


  @if (!hideActions) {

    <div class="ue-actions">

      <button
        type="button"
        class="submit-btn"
        [disabled]="saving"
        (click)="save()"
      >
        {{
          saving
            ? 'Saving...'
            : (
                saved
                  ? 'Save Again'
                  : 'Save'
              )
        }}
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

.question-block {
  display: flex;
  flex-direction: column;

  gap: 10px;

  padding-bottom: 16px;

  border-bottom: 1px solid #eef1f4;
}

.question-block:last-child {
  padding-bottom: 0;

  border-bottom: none;
}

.page-heading {
  margin: 0;

  color: #1e293b;

  font-size: 15px;
  font-weight: 700;
  line-height: 1.4;
}

.radio-column {
  display: flex;
  flex-direction: column;

  gap: 2px;
}

.concerns-grid {
  display: grid;

  grid-template-columns:
    repeat(
      2,
      minmax(0, 1fr)
    );

  gap: 4px 12px;
}

.choice-row {
  display: flex;
  align-items: center;

  gap: 10px;

  padding: 8px 10px;

  border-radius: 8px;

  color: #374151;

  font-size: 13px;

  cursor: pointer;

  transition:
    background-color 0.12s ease;
}

.choice-row:hover {
  background: #f8fafc;
}

.choice-row:has(
  input:checked
) {
  background: #eef6f4;

  color: #1f6b5e;

  font-weight: 500;
}

.choice-row input {
  width: 16px;
  height: 16px;

  flex-shrink: 0;

  accent-color: #269c96;

  cursor: pointer;
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

  box-shadow:
    0 1px 3px
    rgba(31, 132, 127, 0.25);
}

.submit-btn:hover:not(:disabled) {
  background: #1f847f;
}

.submit-btn:disabled {
  background: #b7d4d2;

  border-color: #b7d4d2;

  box-shadow: none;

  cursor: not-allowed;
}

@media (max-width: 700px) {

  .concerns-grid {
    grid-template-columns:
      1fr;
  }
}











questionnaire-end.ts


import {
  Component,
  EventEmitter,
  Output
} from '@angular/core';

@Component({
  selector: 'app-questionnaire-end',
  standalone: true,
  templateUrl:
    './questionnaire-end.html',
  styleUrl:
    './questionnaire-end.css'
})
export class QuestionnaireEnd {

  @Output()
  closeWindow =
    new EventEmitter<void>();

  close(): void {
    this.closeWindow.emit();
  }
}







<div class="thank-you-page">

  <div class="thank-you-card">

    <div class="check-circle">
      ✓
    </div>

    <h1>
      Thank You
    </h1>

    <p>
      Thank you for completing the questionnaire.
    </p>

    <p>
      This information helps us better understand our patients
      and provide the best possible care.
    </p>

    <p>
      Please let a member of our staff know that you have finished.
    </p>

    <button
      type="button"
      class="close-button"
      (click)="close()"
    >
      Close Window
    </button>

  </div>

</div>






:host {
  display: block;
}

.thank-you-page {
  height: 100vh;

  display: flex;
  align-items: center;
  justify-content: center;

  padding: 24px;

  box-sizing: border-box;

  background: #f7f6f3;
}

.thank-you-card {
  width: 100%;
  max-width: 620px;

  padding: 46px 42px;

  box-sizing: border-box;

  background: #ffffff;

  border: 1px solid #dfe5e8;
  border-radius: 10px;

  text-align: center;
}

.check-circle {
  width: 62px;
  height: 62px;

  margin: 0 auto 20px;

  display: flex;
  align-items: center;
  justify-content: center;

  border-radius: 50%;

  background: #e6f4f2;

  color: #00897b;

  font-size: 31px;
  font-weight: 700;
}

.thank-you-card h1 {
  margin: 0 0 20px;

  color: #101a48;

  font-size: 30px;
  font-weight: 700;
}

.thank-you-card p {
  margin: 10px 0;

  color: #56646b;

  font-size: 15px;
  line-height: 1.6;
}

.close-button {
  min-width: 150px;
  height: 44px;

  margin-top: 28px;

  padding: 0 24px;

  border: none;
  border-radius: 6px;

  background: #009688;

  color: #ffffff;

  font-family: inherit;
  font-size: 14px;
  font-weight: 600;

  cursor: pointer;
}

.close-button:hover {
  background: #00796b;
}









Questionnaire,ts
startQuestionnaire(): void {

  if (
    !this.selectedPatient ||
    !this.selectedVisitId ||
    this.selectedQuestionnaires.length === 0
  ) {
    return;
  }

  const query = new URLSearchParams({
    patientId:
      String(
        this.selectedPatient.id
      ),

    visitId:
      String(
        this.selectedVisitId
      ),

    patientName:
      `${this.selectedPatient.fname ?? ''} ${this.selectedPatient.lname ?? ''}`.trim(),

    firstName:
      this.selectedPatient.fname ?? '',

    language:
      this.selectedLanguage,

    selections:
      this.selectedQuestionnaires.join(',')
  });

  window.open(
    `/questionnaire/session?${query.toString()}`,
    '_blank'
  );
}









import { CommonModule } from '@angular/common';
import { HttpClient } from '@angular/common/http';
import {
  ChangeDetectorRef,
  Component,
  OnInit,
  QueryList,
  ViewChildren
} from '@angular/core';
import { ActivatedRoute } from '@angular/router';

import {
  FirstVisitQuestionnaire
} from '../first-visit-questionnaire/first-visit-questionnaire';

import {
  BaselineWho,
  HistoryQuestionnaire
} from '../history-questionnaire/history-questionnaire';

import {
  HipQuestionnaire
} from '../hip-questionnaire/hip-questionnaire';

import {
  PodciQuestionnaire,
  PodciVariant
} from '../podci-questionnaire/podci-questionnaire';

import {
  QuestionnaireClose
} from '../questionnaire-close/questionnaire-close';

import {
  QuestionnaireEnd
} from '../questionnaire-end/questionnaire-end';

import {
  QuestionnairePatientInfo
} from '../questionnaire-patient-info/questionnaire-patient-info';

import {
  QuestionnaireProgress
} from '../questionnaire-progress/questionnaire-progress';

import {
  QuestionnaireWelcome
} from '../questionnaire-welcome/questionnaire-welcome';

import {
  SportsQuestionnaire
} from '../sports-questionnaire/sports-questionnaire';

import {
  UeHistoryQuestionnaire
} from '../ue-history-questionnaire/ue-history-questionnaire';


interface AssembledQuestionnaire {
  code: string;
  visitId: number;
}


interface QuestionnaireStep {
  key: string;
  title: string;
  codes: string[];
  visitId: number | null;
}


interface PatientInfo {
  id: number;
  fname?: string | null;
  lname?: string | null;
  mrn?: string | null;
  dob?: string | null;
  gender?: string | null;
  sex?: string | null;
}


interface QuestionnaireProgressValue {
  answered: number;
  total: number;
}


interface QuestionnaireProgressItem
extends QuestionnaireProgressValue {
  key: string;
  name: string;
  completed: boolean;
}


interface SavableQuestionnaire {
  save(): void;
  validate?(): string | null;
  getProgress?(): QuestionnaireProgressValue;
}


type SaveAction =
  | 'next'
  | 'close'
  | 'submit'
  | null;


const HISTORY_CODES =
  new Set<string>([
    'NEW_HISTORY',
    'HISTORY',
    'HISTORY_GAIT',
    'HISTORY_CONCERNS'
  ]);


@Component({
  selector: 'app-questionnaire-session',
  standalone: true,
  imports: [
    CommonModule,
    QuestionnaireWelcome,
    QuestionnaireClose,
    QuestionnaireEnd,
    QuestionnairePatientInfo,
    QuestionnaireProgress,
    FirstVisitQuestionnaire,
    HistoryQuestionnaire,
    HipQuestionnaire,
    SportsQuestionnaire,
    PodciQuestionnaire,
    UeHistoryQuestionnaire
  ],
  templateUrl: './questionnaire-session.html',
  styleUrl: './questionnaire-session.css'
})
export class QuestionnaireSession
implements OnInit {

  patientId:
    number | null = null;

  visitId:
    number | null = null;


  patientDisplayName = '';

  firstName = '';

  patientDob = '';

  patientSex = '';

  patientMrn = '';


  languageCode:
    'en' | 'sp' = 'en';

  private language = 'en';

  private selections:
    string[] = [];

  private assembledList:
    AssembledQuestionnaire[] = [];


  assembling = false;

  assembleError = '';

  saving = false;

  saveError = '';


  steps:
    QuestionnaireStep[] = [];

  currentIndex = 0;


  progressItems:
    QuestionnaireProgressItem[] = [];


  showWelcome = true;

  showClosePopup = false;

  showThankYou = false;


  baseline:
    BaselineWho | null = null;


  private pendingSaveCount = 0;

  private saveFailed = false;

  private saveAction:
    SaveAction = null;


  @ViewChildren('sectionCmp')
  private sectionCmps!:
    QueryList<SavableQuestionnaire>;


  constructor(
    private http:
      HttpClient,

    private route:
      ActivatedRoute,

    private cdr:
      ChangeDetectorRef
  ) {}


  ngOnInit(): void {

    const params =
      this.route
        .snapshot
        .queryParamMap;


    this.patientId =
      params.get('patientId')
        ? Number(
            params.get('patientId')
          )
        : null;


    this.visitId =
      params.get('visitId')
        ? Number(
            params.get('visitId')
          )
        : null;


    this.patientDisplayName =
      params.get('patientName') ??
      '';


    this.firstName =
      params.get('firstName') ??
      '';


    this.language =
      params.get('language') ??
      'en';


    this.languageCode =
      this.language === 'sp'
        ? 'sp'
        : 'en';


    this.selections =
      (
        params.get('selections') ??
        ''
      )
        .split(',')
        .map(
          value =>
            value.trim()
        )
        .filter(Boolean);


    this.loadPatient();

    this.assemble();
  }


  private loadPatient():
    void {

    if (
      !this.patientId
    ) {
      return;
    }


    this.http
      .get<PatientInfo>(
        `/api/patients/${this.patientId}`
      )
      .subscribe({

        next:
          patient => {

            const fullName =
              `${patient.fname ?? ''} ${patient.lname ?? ''}`
                .trim();


            if (
              fullName
            ) {

              this.patientDisplayName =
                fullName;
            }


            if (
              patient.fname
            ) {

              this.firstName =
                patient.fname;
            }


            this.patientDob =
              patient.dob ??
              '';


            this.patientSex =
              patient.gender ??
              patient.sex ??
              '';


            this.patientMrn =
              patient.mrn ??
              '';


            this.cdr
              .markForCheck();
          },


        error:
          error => {

            console.error(
              'Unable to load questionnaire patient information:',
              error
            );
          }
      });
  }


  private assemble():
    void {

    if (
      !this.patientId ||
      !this.visitId ||
      this.selections.length === 0
    ) {

      this.assembleError =
        'Missing patient, visit, or questionnaire selection.';

      return;
    }


    this.assembling =
      true;

    this.assembleError =
      '';


    this.http
      .post<
        AssembledQuestionnaire[]
      >(
        `/api/patients/${this.patientId}/questionnaire-picker/assemble`,
        {
          visitId:
            this.visitId,

          selections:
            this.selections,

          language:
            this.language
        }
      )
      .subscribe({

        next:
          assembled => {

            this.assembledList =
              assembled ?? [];


            this.steps =
              this.buildSteps(
                this.assembledList
              );


            this.progressItems =
              this.steps.map(
                step => ({
                  key:
                    step.key,

                  name:
                    step.title,

                  answered:
                    0,

                  total:
                    0,

                  completed:
                    false
                })
              );


            this.currentIndex =
              0;


            this.assembling =
              false;


            this.cdr
              .detectChanges();
          },


        error:
          error => {

            console.error(
              'Unable to assemble questionnaires:',
              error
            );


            this.assembling =
              false;


            this.assembleError =
              'Unable to prepare the questionnaires. Please try again.';


            this.cdr
              .detectChanges();
          }
      });
  }


  private buildSteps(
    assembled:
      AssembledQuestionnaire[]
  ): QuestionnaireStep[] {

    const steps:
      QuestionnaireStep[] = [];


    let historyAdded =
      false;


    for (
      const item of assembled
    ) {

      if (
        HISTORY_CODES.has(
          item.code
        )
      ) {

        if (
          !historyAdded
        ) {

          const historyItems =
            assembled.filter(
              questionnaire =>
                HISTORY_CODES.has(
                  questionnaire.code
                )
            );


          steps.push({
            key:
              'HISTORY',

            title:
              'History',

            codes:
              historyItems.map(
                questionnaire =>
                  questionnaire.code
              ),

            visitId:
              historyItems[0]
                ?.visitId ??
              this.visitId
          });


          historyAdded =
            true;
        }


        continue;
      }


      if (
        item.code === 'HIP'
      ) {

        steps.push({
          key:
            'HIP',

          title:
            'Hip',

          codes: [
            'HIP'
          ],

          visitId:
            item.visitId
        });


        continue;
      }


      if (
        item.code === 'SPORTS'
      ) {

        steps.push({
          key:
            'SPORTS',

          title:
            'Sport',

          codes: [
            'SPORTS'
          ],

          visitId:
            item.visitId
        });


        continue;
      }


      if (
        item.code === 'PODCI_CH'
      ) {

        steps.push({
          key:
            'PODCI_CH',

          title:
            'Children',

          codes: [
            'PODCI_CH'
          ],

          visitId:
            item.visitId
        });


        continue;
      }


      if (
        item.code === 'PODCI_AP'
      ) {

        steps.push({
          key:
            'PODCI_AP',

          title:
            'PODCI Parent',

          codes: [
            'PODCI_AP'
          ],

          visitId:
            item.visitId
        });


        continue;
      }


      if (
        item.code === 'PODCI_AS'
      ) {

        steps.push({
          key:
            'PODCI_AS',

          title:
            'Adult',

          codes: [
            'PODCI_AS'
          ],

          visitId:
            item.visitId
        });


        continue;
      }


      if (
        item.code === 'UE_HISTORY'
      ) {

        steps.push({
          key:
            'UE_HISTORY',

          title:
            'UE History',

          codes: [
            'UE_HISTORY'
          ],

          visitId:
            item.visitId
        });
      }
    }


    return steps;
  }


  retry():
    void {

    this.assemble();
  }


  get questionnaireNames():
    string[] {

    return (
      this.steps.map(
        step =>
          step.title
      )
    );
  }


  get currentStep():
    QuestionnaireStep | null {

    return (
      this.steps[
        this.currentIndex
      ] ??
      null
    );
  }


  get isLastQuestionnaire():
    boolean {

    if (
      this.steps.length === 0
    ) {
      return false;
    }


    return (
      this.currentIndex ===
      this.steps.length - 1
    );
  }


  get remainingCount():
    number {

    return Math.max(
      this.steps.length -
        this.currentIndex -
        1,
      0
    );
  }


  get progressText():
    string {

    return (
      `Questionnaire ${
        this.currentIndex + 1
      } of ${
        this.steps.length
      }`
    );
  }


  get showFirstVisit():
    boolean {

    return (
      this.hasCurrentCode(
        'NEW_HISTORY'
      )
    );
  }


  get showWho():
    boolean {

    return (
      this.hasCurrentCode(
        'HISTORY'
      )
    );
  }


  get showGeneral():
    boolean {

    return (
      this.hasCurrentCode(
        'HISTORY'
      ) ||
      this.hasCurrentCode(
        'NEW_HISTORY'
      )
    );
  }


  get showGait():
    boolean {

    return (
      this.hasCurrentCode(
        'HISTORY_GAIT'
      )
    );
  }


  get showConcerns():
    boolean {

    return (
      this.hasCurrentCode(
        'HISTORY_CONCERNS'
      )
    );
  }


  get historyVisitId():
    number | null {

    const history =
      this.assembledList
        .find(
          item =>
            HISTORY_CODES.has(
              item.code
            )
        );


    return (
      history?.visitId ??
      this.visitId
    );
  }


  hasCurrentCode(
    code:
      string
  ): boolean {

    return (
      this.currentStep
        ?.codes
        .includes(
          code
        ) ??
      false
    );
  }


  podciVariant(
    code:
      string
  ): PodciVariant {

    if (
      code === 'PODCI_AP'
    ) {
      return 'AP';
    }


    if (
      code === 'PODCI_AS'
    ) {
      return 'AS';
    }


    return 'CH';
  }


  startQuestionnaires():
    void {

    this.showWelcome =
      false;


    this.saveError =
      '';


    this.cdr
      .detectChanges();


    this.scheduleProgressRefresh();


    this.scrollQuestionArea();
  }


  onBaselineChange(
    baseline:
      BaselineWho
  ): void {

    this.baseline =
      baseline;


    this.scheduleProgressRefresh();
  }


  scheduleProgressRefresh():
    void {

    setTimeout(
      () => {

        this.refreshCurrentProgress();

      },
      0
    );
  }


  refreshCurrentProgress():
    void {

    if (
      !this.currentStep ||
      !this.sectionCmps
    ) {
      return;
    }


    let answered =
      0;

    let total =
      0;


    const components =
      this.sectionCmps
        .toArray();


    for (
      const component of components
    ) {

      const progress =
        component
          .getProgress?.();


      if (
        !progress
      ) {
        continue;
      }


      answered +=
        progress.answered;


      total +=
        progress.total;
    }


    const item =
      this.progressItems
        .find(
          progress =>
            progress.key ===
            this.currentStep?.key
        );


    if (
      !item
    ) {
      return;
    }


    item.answered =
      answered;


    item.total =
      total;


    this.progressItems = [
      ...this.progressItems
    ];


    this.cdr
      .markForCheck();
  }


  nextQuestionnaire():
    void {

    if (
      this.saving ||
      !this.currentStep
    ) {
      return;
    }


    this.saveCurrent(
      'next'
    );
  }


  requestSaveAndClose():
    void {

    if (
      this.saving ||
      !this.currentStep
    ) {
      return;
    }


    if (
      this.remainingCount >
      0
    ) {

      this.showClosePopup =
        true;

      return;
    }


    this.saveCurrent(
      'close'
    );
  }


  continueQuestionnaire():
    void {

    this.showClosePopup =
      false;
  }


  confirmSaveAndClose():
    void {

    this.showClosePopup =
      false;


    this.saveCurrent(
      'close'
    );
  }


  submitQuestionnaires():
    void {

    if (
      this.saving ||
      !this.currentStep ||
      !this.isLastQuestionnaire
    ) {
      return;
    }


    this.saveCurrent(
      'submit'
    );
  }


  private saveCurrent(
    action:
      SaveAction
  ): void {

    const components =
      this.sectionCmps
        .toArray();


    if (
      components.length === 0
    ) {

      this.afterSuccessfulSave(
        action
      );

      return;
    }


    for (
      const component of components
    ) {

      const validation =
        component
          .validate?.() ??
        null;


      if (
        validation
      ) {

        this.saveError =
          validation;


        this.cdr
          .detectChanges();


        return;
      }
    }


    this.saving =
      true;


    this.saveError =
      '';


    this.saveAction =
      action;


    this.pendingSaveCount =
      components.length;


    this.saveFailed =
      false;


    for (
      const component of components
    ) {

      component.save();
    }
  }


  onSectionSaved():
    void {

    if (
      !this.saving
    ) {
      return;
    }


    this.pendingSaveCount--;


    this.finishCurrentSave();
  }


  onSectionSaveFailed(
    message?:
      string
  ): void {

    if (
      !this.saving
    ) {
      return;
    }


    this.saveFailed =
      true;


    this.saveError =
      message ||
      'Unable to save this questionnaire. Please try again.';


    this.pendingSaveCount--;


    this.finishCurrentSave();
  }


  private finishCurrentSave():
    void {

    if (
      this.pendingSaveCount >
      0
    ) {
      return;
    }


    this.saving =
      false;


    if (
      this.saveFailed
    ) {

      this.saveAction =
        null;


      this.cdr
        .detectChanges();


      return;
    }


    const action =
      this.saveAction;


    this.saveAction =
      null;


    this.afterSuccessfulSave(
      action
    );
  }


  private afterSuccessfulSave(
    action:
      SaveAction
  ): void {

    this.refreshCurrentProgress();


    if (
      this.currentStep
    ) {

      const item =
        this.progressItems
          .find(
            progress =>
              progress.key ===
              this.currentStep?.key
          );


      if (
        item
      ) {

        item.completed =
          true;


        this.progressItems = [
          ...this.progressItems
        ];
      }
    }


    if (
      action === 'next'
    ) {

      if (
        this.currentIndex <
        this.steps.length - 1
      ) {

        this.currentIndex++;


        this.saveError =
          '';


        this.cdr
          .detectChanges();


        this.scheduleProgressRefresh();


        this.scrollQuestionArea();
      }


      return;
    }


    if (
      action === 'close'
    ) {

      window.close();

      return;
    }


    if (
      action === 'submit'
    ) {

      this.recordTracking();


      this.showThankYou =
        true;


      this.saveError =
        '';


      this.cdr
        .detectChanges();
    }
  }


  private recordTracking():
    void {

    if (
      !this.patientId ||
      this.assembledList.length === 0
    ) {
      return;
    }


    this.http
      .post(
        `/api/patients/${this.patientId}/questionnaire-tracking`,
        this.assembledList
      )
      .subscribe({

        error:
          error => {

            console.error(
              'Unable to record questionnaire tracking:',
              error
            );
          }
      });
  }


  closeWindow():
    void {

    window.close();
  }


  private scrollQuestionArea():
    void {

    setTimeout(
      () => {

        document
          .querySelector(
            '.question-scroll'
          )
          ?.scrollTo({
            top:
              0,

            behavior:
              'smooth'
          });

      },
      0
    );
  }
}

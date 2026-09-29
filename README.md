import {
  Component,
  EventEmitter,
  Input,
  Output
} from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-questionnaire-welcome',
  standalone: true,
  imports: [CommonModule],
  templateUrl: './questionnaire-welcome.html',
  styleUrl: './questionnaire-welcome.css'
})
export class QuestionnaireWelcome {
  @Input() patientName = '';
  @Input() questionnaireNames: string[] = [];

  @Output() startQuestionnaire =
    new EventEmitter<void>();

  start(): void {
    this.startQuestionnaire.emit();
  }
}






<div class="welcome-page">

  <div class="welcome-card">

    <h1>
      Welcome to the duPont Hospital for Children Gait Lab
    </h1>

    <p class="welcome-message">
      In order to best understand our patients,
      we need your help completing the following questionnaires.
    </p>

    @if (patientName) {
      <p class="patient-name">
        Patient:
        <strong>{{ patientName }}</strong>
      </p>
    }

    <div class="questionnaire-list">

      <h2>
        The following questionnaires need to be completed:
      </h2>

      <ol>
        @for (
          questionnaire of questionnaireNames;
          track questionnaire
        ) {
          <li>
            {{ questionnaire }}
          </li>
        }
      </ol>

    </div>

    <p class="help-text">
      If you have any questions while completing
      the questionnaire, please ask a member of our staff for help.
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








.welcome-page {
  display: flex;
  justify-content: center;
  padding: 36px 24px;
}

.welcome-card {
  width: 100%;
  max-width: 820px;
  background: #ffffff;
  border: 1px solid #dfe5e8;
  border-radius: 10px;
  padding: 34px 40px;
  box-sizing: border-box;
}

.welcome-card h1 {
  margin: 0 0 18px;
  font-size: 25px;
  font-weight: 700;
  color: #263238;
}

.welcome-message {
  margin: 0 0 16px;
  font-size: 15px;
  line-height: 1.6;
  color: #4f5b62;
}

.patient-name {
  margin: 0 0 24px;
  font-size: 14px;
  color: #37474f;
}

.questionnaire-list {
  background: #f7f9fa;
  border: 1px solid #e1e6e9;
  border-radius: 8px;
  padding: 22px 26px;
}

.questionnaire-list h2 {
  margin: 0 0 12px;
  font-size: 17px;
  color: #263238;
}

.questionnaire-list ol {
  margin: 0;
  padding-left: 24px;
}

.questionnaire-list li {
  padding: 5px 0;
  font-size: 15px;
  color: #37474f;
}

.help-text {
  margin: 22px 0 0;
  font-size: 14px;
  line-height: 1.6;
  color: #607078;
}

.welcome-actions {
  display: flex;
  justify-content: flex-end;
  margin-top: 28px;
}

.start-button {
  min-width: 110px;
  height: 40px;
  padding: 0 24px;
  border: none;
  border-radius: 6px;
  background: #009688;
  color: white;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
}

.start-button:hover {
  background: #00796b;
}









import {
  Component,
  EventEmitter,
  Input,
  Output
} from '@angular/core';

@Component({
  selector: 'app-questionnaire-close',
  standalone: true,
  templateUrl: './questionnaire-close.html',
  styleUrl: './questionnaire-close.css'
})
export class QuestionnaireClose {
  @Input() remainingCount = 0;
  @Input() saving = false;

  @Output() continueQuestionnaire =
    new EventEmitter<void>();

  @Output() saveAndClose =
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
      There
      @if (remainingCount === 1) {
        is 1 questionnaire
      } @else {
        are {{ remainingCount }} questionnaires
      }
      left to complete.
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
  background: rgba(0, 0, 0, 0.35);
}

.close-modal {
  width: 100%;
  max-width: 500px;
  background: #ffffff;
  border-radius: 10px;
  padding: 28px;
  box-sizing: border-box;
  box-shadow: 0 14px 40px rgba(0, 0, 0, 0.2);
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
  background: white;
  color: #455a64;
  border: 1px solid #b8c3c8;
}

.close-button {
  border: none;
  background: #009688;
  color: white;
}

button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}









import {
  Component,
  EventEmitter,
  Output
} from '@angular/core';

@Component({
  selector: 'app-questionnaire-thank-you',
  standalone: true,
  templateUrl: './questionnaire-thank-you.html',
  styleUrl: './questionnaire-thank-you.css'
})
export class QuestionnaireThankYou {
  @Output() closeWindow =
    new EventEmitter<void>();

  close(): void {
    this.closeWindow.emit();
  }
}









<div class="thank-you-page">

  <div class="thank-you-card">

    <div class="check">
      ✓
    </div>

    <h1>
      Thank You
    </h1>

    <p>
      Thank you for completing the questionnaire.
    </p>

    <p>
      This information helps us better understand
      our patients and provide the best possible care.
    </p>

    <p>
      Please let a member of our staff know
      that you have finished.
    </p>

    <button
      type="button"
      (click)="close()"
    >
      Close Window
    </button>

  </div>

</div>









.thank-you-page {
  display: flex;
  justify-content: center;
  padding: 70px 24px;
}

.thank-you-card {
  width: 100%;
  max-width: 620px;
  padding: 45px 40px;
  background: #ffffff;
  border: 1px solid #dfe5e8;
  border-radius: 10px;
  text-align: center;
}

.check {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 58px;
  height: 58px;
  margin: 0 auto 18px;
  border-radius: 50%;
  background: #e3f3f1;
  color: #00897b;
  font-size: 30px;
  font-weight: 700;
}

.thank-you-card h1 {
  margin: 0 0 18px;
  color: #263238;
  font-size: 28px;
}

.thank-you-card p {
  margin: 8px 0;
  color: #58666d;
  line-height: 1.6;
  font-size: 14px;
}

.thank-you-card button {
  margin-top: 28px;
  min-width: 130px;
  height: 40px;
  padding: 0 22px;
  border: none;
  border-radius: 6px;
  background: #009688;
  color: white;
  font-weight: 600;
  cursor: pointer;
}








import { CommonModule } from '@angular/common';
import {
  ChangeDetectorRef,
  Component,
  OnInit,
  QueryList,
  ViewChildren
} from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { ActivatedRoute } from '@angular/router';

import { Header } from '../../header/header';

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
  SportsQuestionnaire
} from '../sports-questionnaire/sports-questionnaire';

import {
  PodciQuestionnaire,
  PodciVariant
} from '../podci-questionnaire/podci-questionnaire';

import {
  UeHistoryQuestionnaire
} from '../ue-history-questionnaire/ue-history-questionnaire';

import {
  QuestionnaireWelcome
} from '../questionnaire-welcome/questionnaire-welcome';

import {
  QuestionnaireClose
} from '../questionnaire-close/questionnaire-close';

import {
  QuestionnaireThankYou
} from '../questionnaire-thank-you/questionnaire-thank-you';


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


interface SavableQuestionnaire {
  save(): void;
  validate?(): string | null;
}


type SaveAction =
  | 'next'
  | 'close'
  | 'submit'
  | null;


const HISTORY_CODES = new Set([
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
    Header,
    QuestionnaireWelcome,
    QuestionnaireClose,
    QuestionnaireThankYou,
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
export class QuestionnaireSession implements OnInit {

  patientId: number | null = null;
  visitId: number | null = null;

  patientDisplayName = '';
  firstName = '';

  languageCode:
    'en' | 'sp' = 'en';

  private language = 'en';

  private selections:
    string[] = [];


  assembling = false;
  assembleError = '';

  private assembledList:
    AssembledQuestionnaire[] = [];


  steps:
    QuestionnaireStep[] = [];

  currentIndex = 0;


  showWelcome = true;
  showClosePopup = false;
  showThankYou = false;


  baseline:
    BaselineWho | null = null;


  saving = false;
  saveError = '';

  private pendingSaveCount = 0;

  private saveFailed = false;

  private saveAction:
    SaveAction = null;


  @ViewChildren('sectionCmp')
  private sectionCmps!:
    QueryList<SavableQuestionnaire>;


  constructor(
    private http: HttpClient,
    private route: ActivatedRoute,
    private cdr: ChangeDetectorRef
  ) {}


  ngOnInit(): void {

    const params =
      this.route.snapshot.queryParamMap;

    this.patientId =
      params.get('patientId')
        ? Number(params.get('patientId'))
        : null;

    this.visitId =
      params.get('visitId')
        ? Number(params.get('visitId'))
        : null;

    this.patientDisplayName =
      params.get('patientName') ?? '';

    this.firstName =
      params.get('firstName') ?? '';

    this.language =
      params.get('language') ?? 'en';

    this.languageCode =
      this.language === 'sp'
        ? 'sp'
        : 'en';

    this.selections =
      (params.get('selections') ?? '')
        .split(',')
        .filter(Boolean);

    this.assemble();
  }


  private assemble(): void {

    if (
      !this.patientId ||
      !this.visitId ||
      this.selections.length === 0
    ) {
      this.assembleError =
        'Missing patient, visit, or questionnaire selection.';

      return;
    }

    this.assembling = true;
    this.assembleError = '';

    this.http
      .post<AssembledQuestionnaire[]>(
        `/api/patients/${this.patientId}/questionnaire-picker/assemble`,
        {
          visitId: this.visitId,
          selections: this.selections,
          language: this.language
        }
      )
      .subscribe({

        next: assembled => {

          this.assembledList =
            assembled;

          this.steps =
            this.buildSteps(assembled);

          this.assembling = false;

          this.cdr.detectChanges();
        },

        error: error => {

          console.error(
            'Unable to assemble questionnaires:',
            error
          );

          this.assembling = false;

          this.assembleError =
            'Unable to prepare the questionnaires. Please try again.';

          this.cdr.detectChanges();
        }
      });
  }


  private buildSteps(
    assembled:
      AssembledQuestionnaire[]
  ): QuestionnaireStep[] {

    const steps:
      QuestionnaireStep[] = [];

    let historyAdded = false;


    for (const item of assembled) {

      if (
        HISTORY_CODES.has(item.code)
      ) {

        if (!historyAdded) {

          const historyItems =
            assembled.filter(
              questionnaire =>
                HISTORY_CODES.has(
                  questionnaire.code
                )
            );

          steps.push({
            key: 'HISTORY',
            title: 'History',
            codes:
              historyItems.map(
                questionnaire =>
                  questionnaire.code
              ),
            visitId:
              historyItems[0]?.visitId ??
              this.visitId
          });

          historyAdded = true;
        }

        continue;
      }


      if (item.code === 'HIP') {

        steps.push({
          key: 'HIP',
          title: 'Hip',
          codes: ['HIP'],
          visitId: item.visitId
        });

        continue;
      }


      if (item.code === 'SPORTS') {

        steps.push({
          key: 'SPORTS',
          title: 'Sports',
          codes: ['SPORTS'],
          visitId: item.visitId
        });

        continue;
      }


      if (item.code === 'PODCI_CH') {

        steps.push({
          key: 'PODCI_CH',
          title: 'PODCI (Child)',
          codes: ['PODCI_CH'],
          visitId: item.visitId
        });

        continue;
      }


      if (item.code === 'PODCI_AP') {

        steps.push({
          key: 'PODCI_AP',
          title:
            'PODCI (Adolescent Parent-reported)',
          codes: ['PODCI_AP'],
          visitId: item.visitId
        });

        continue;
      }


      if (item.code === 'PODCI_AS') {

        steps.push({
          key: 'PODCI_AS',
          title:
            'PODCI (Adolescent Self-reported)',
          codes: ['PODCI_AS'],
          visitId: item.visitId
        });

        continue;
      }


      if (item.code === 'UE_HISTORY') {

        steps.push({
          key: 'UE_HISTORY',
          title: 'UE History',
          codes: ['UE_HISTORY'],
          visitId: item.visitId
        });
      }
    }

    return steps;
  }


  retry(): void {
    this.assemble();
  }


  get questionnaireNames():
    string[] {

    return this.steps.map(
      step => step.title
    );
  }


  get currentStep():
    QuestionnaireStep | null {

    return (
      this.steps[this.currentIndex] ??
      null
    );
  }


  get isLastQuestionnaire():
    boolean {

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
      } of ${this.steps.length}`
    );
  }


  startQuestionnaires(): void {

    this.showWelcome = false;

    this.scrollTop();
  }


  hasCurrentCode(
    code: string
  ): boolean {

    return (
      this.currentStep?.codes.includes(
        code
      ) ?? false
    );
  }


  get showFirstVisit():
    boolean {

    return this.hasCurrentCode(
      'NEW_HISTORY'
    );
  }


  get showWho():
    boolean {

    return this.hasCurrentCode(
      'HISTORY'
    );
  }


  get showGeneral():
    boolean {

    return (
      this.hasCurrentCode('HISTORY') ||
      this.hasCurrentCode('NEW_HISTORY')
    );
  }


  get showGait():
    boolean {

    return this.hasCurrentCode(
      'HISTORY_GAIT'
    );
  }


  get showConcerns():
    boolean {

    return this.hasCurrentCode(
      'HISTORY_CONCERNS'
    );
  }


  get historyVisitId():
    number | null {

    const history =
      this.assembledList.find(
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


  podciVariant(
    code: string
  ): PodciVariant {

    if (code === 'PODCI_AP') {
      return 'AP';
    }

    if (code === 'PODCI_AS') {
      return 'AS';
    }

    return 'CH';
  }


  onBaselineChange(
    baseline: BaselineWho
  ): void {

    this.baseline =
      baseline;
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
      this.isLastQuestionnaire
        ? 'submit'
        : 'next'
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
      this.remainingCount > 0
    ) {

      this.showClosePopup =
        true;

      return;
    }

    this.saveCurrent('close');
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

    this.saveCurrent('close');
  }


  submitQuestionnaires():
    void {

    if (
      this.saving ||
      !this.isLastQuestionnaire
    ) {
      return;
    }

    this.saveCurrent('submit');
  }


  private saveCurrent(
    action: SaveAction
  ): void {

    const components =
      this.sectionCmps.toArray();


    if (components.length === 0) {

      this.afterSuccessfulSave(
        action
      );

      return;
    }


    for (
      const component of components
    ) {

      const validation =
        component.validate?.() ??
        null;

      if (validation) {

        this.saveError =
          validation;

        this.cdr.detectChanges();

        return;
      }
    }


    this.saving = true;

    this.saveError = '';

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

    if (!this.saving) {
      return;
    }

    this.pendingSaveCount--;

    this.finishCurrentSave();
  }


  onSectionSaveFailed(
    message?: string
  ): void {

    if (!this.saving) {
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
      this.pendingSaveCount > 0
    ) {
      return;
    }


    this.saving = false;


    if (
      this.saveFailed
    ) {

      this.saveAction =
        null;

      this.cdr.detectChanges();

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
    action: SaveAction
  ): void {

    if (
      action === 'next'
    ) {

      if (
        this.currentIndex <
        this.steps.length - 1
      ) {

        this.currentIndex++;

        this.saveError = '';

        this.cdr.detectChanges();

        this.scrollTop();
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

      this.saveError = '';

      this.cdr.detectChanges();

      this.scrollTop();
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
        error: error =>
          console.error(
            'Unable to record questionnaire tracking:',
            error
          )
      });
  }


  closeWindow():
    void {

    window.close();
  }


  private scrollTop():
    void {

    window.scrollTo({
      top: 0,
      behavior: 'smooth'
    });
  }
}










<div class="session-page">

  <app-header></app-header>

  <main class="session-content">


    @if (assembling) {

      <div class="status-card">

        <h2>
          Preparing Questionnaires
        </h2>

        <p>
          Please wait...
        </p>

      </div>

    } @else if (assembleError) {

      <div class="status-card">

        <h2>
          Something went wrong
        </h2>

        <div class="error-message">
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

      <div class="questionnaire-header">

        <div>

          <div class="progress">
            {{ progressText }}
          </div>

          <h1>
            {{ currentStep.title }}
          </h1>

        </div>

      </div>


      @if (saveError) {

        <div class="error-message">
          {{ saveError }}
        </div>

      }


      <div class="questionnaire-body">


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

              (baselineChange)="
                onBaselineChange($event)
              "

              (saveComplete)="
                onSectionSaved()
              "

              (saveFailed)="
                onSectionSaveFailed($event)
              "
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

            (saveComplete)="
              onSectionSaved()
            "

            (saveFailed)="
              onSectionSaveFailed($event)
            "
          >
          </app-history-questionnaire>

        }


        @else if (
          currentStep.key ===
          'HIP'
        ) {

          <app-hip-questionnaire
            #sectionCmp

            [patientId]="patientId"

            [visitId]="currentStep.visitId"

            [hideActions]="true"

            (saveComplete)="
              onSectionSaved()
            "

            (saveFailed)="
              onSectionSaveFailed($event)
            "
          >
          </app-hip-questionnaire>

        }


        @else if (
          currentStep.key ===
          'SPORTS'
        ) {

          <app-sports-questionnaire
            #sectionCmp

            [patientId]="patientId"

            [visitId]="currentStep.visitId"

            [hideActions]="true"

            (saveComplete)="
              onSectionSaved()
            "

            (saveFailed)="
              onSectionSaveFailed($event)
            "
          >
          </app-sports-questionnaire>

        }


        @else if (
          currentStep.key ===
          'PODCI_CH' ||
          currentStep.key ===
          'PODCI_AP' ||
          currentStep.key ===
          'PODCI_AS'
        ) {

          <app-podci-questionnaire
            #sectionCmp

            [patientId]="patientId"

            [visitId]="currentStep.visitId"

            [firstName]="firstName"

            [language]="languageCode"

            [variant]="
              podciVariant(
                currentStep.key
              )
            "

            [hideActions]="true"

            (saveComplete)="
              onSectionSaved()
            "

            (saveFailed)="
              onSectionSaveFailed($event)
            "
          >
          </app-podci-questionnaire>

        }


        @else if (
          currentStep.key ===
          'UE_HISTORY'
        ) {

          <app-ue-history-questionnaire
            #sectionCmp

            [patientId]="patientId"

            [visitId]="currentStep.visitId"

            [hideActions]="true"

            (saveComplete)="
              onSectionSaved()
            "

            (saveFailed)="
              onSectionSaveFailed($event)
            "
          >
          </app-ue-history-questionnaire>

        }

      </div>


      <div class="questionnaire-actions">

        <button
          type="button"
          class="secondary-button"
          [disabled]="saving"
          (click)="requestSaveAndClose()"
        >

          @if (
            saving &&
            isLastQuestionnaire
          ) {
            Saving...
          } @else {
            Save and Close
          }

        </button>


        @if (!isLastQuestionnaire) {

          <button
            type="button"
            class="primary-button"
            [disabled]="saving"
            (click)="nextQuestionnaire()"
          >

            @if (saving) {
              Saving...
            } @else {
              Next Questionnaire
            }

          </button>

        } @else {

          <button
            type="button"
            class="primary-button"
            [disabled]="saving"
            (click)="submitQuestionnaires()"
          >

            @if (saving) {
              Submitting...
            } @else {
              Submit
            }

          </button>

        }

      </div>

    }

  </main>


  @if (showClosePopup) {

    <app-questionnaire-close
      [remainingCount]="remainingCount"
      [saving]="saving"

      (continueQuestionnaire)="
        continueQuestionnaire()
      "

      (saveAndClose)="
        confirmSaveAndClose()
      "
    >
    </app-questionnaire-close>

  }

</div>










.session-page {
  min-height: 100vh;
  background: #f5f7f8;
}

.session-content {
  width: 100%;
  max-width: 1000px;
  margin: 0 auto;
  padding: 28px 30px 70px;
  box-sizing: border-box;
}

.status-card {
  max-width: 700px;
  margin: 50px auto;
  padding: 30px;
  background: #ffffff;
  border: 1px solid #dfe5e8;
  border-radius: 10px;
}

.questionnaire-header {
  margin-bottom: 16px;
}

.progress {
  margin-bottom: 4px;
  color: #728087;
  font-size: 12px;
  font-weight: 600;
  text-transform: uppercase;
}

.questionnaire-header h1 {
  margin: 0;
  color: #263238;
  font-size: 25px;
}

.questionnaire-body {
  background: #ffffff;
  border: 1px solid #dfe5e8;
  border-radius: 10px;
  padding: 26px;
}

.error-message {
  margin-bottom: 16px;
  padding: 12px 15px;
  border: 1px solid #e3a9a9;
  border-radius: 6px;
  background: #fff3f3;
  color: #a12626;
  font-size: 13px;
}

.questionnaire-actions {
  position: sticky;
  bottom: 0;
  z-index: 30;
  display: flex;
  justify-content: flex-end;
  gap: 10px;
  padding: 18px 0;
  margin-top: 18px;
  background: #f5f7f8;
}

.primary-button,
.secondary-button {
  min-height: 40px;
  padding: 0 20px;
  border-radius: 6px;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
}

.primary-button {
  border: none;
  background: #009688;
  color: white;
}

.primary-button:hover {
  background: #00796b;
}

.secondary-button {
  border: 1px solid #aebbc0;
  background: white;
  color: #455a64;
}

button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

@media (max-width: 700px) {
  .session-content {
    padding: 18px 14px 50px;
  }

  .questionnaire-body {
    padding: 18px;
  }

  .questionnaire-actions {
    display: grid;
    grid-template-columns: 1fr 1fr;
  }
}












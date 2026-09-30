<div class="welcome-page">

  <div class="welcome-card">

    <img
      src="assets/questionnaire.jpeg"
      alt="Gait and Motion Analysis Laboratory Patient Questionnaire"
      class="welcome-banner"
    />

    <div class="welcome-content">

      <p class="intro-text">
        In order to better understand our patients,
        we need your help with some history about
        <strong>{{ patientName }}</strong>.
      </p>

      <div class="questionnaire-section">

        <p class="section-title">
          The following questionnaire needs to be completed:
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
        These questions will be asked on each visit so we can stay
        up to date with you.
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

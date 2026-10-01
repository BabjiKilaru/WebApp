import { CommonModule } from '@angular/common';
import {
  Component,
  Input
} from '@angular/core';

export interface QuestionnaireProgressValue {
  answered: number;
  total: number;
}

export interface QuestionnaireProgressItem
extends QuestionnaireProgressValue {
  key: string;
  name: string;
  completed: boolean;
}

@Component({
  selector: 'app-questionnaire-progress',
  standalone: true,
  imports: [
    CommonModule
  ],
  templateUrl: './questionnaire-progress.html',
  styleUrl: './questionnaire-progress.css'
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

import { DatePipe, DecimalPipe, NgTemplateOutlet } from '@angular/common';
import {
    ChangeDetectorRef, Component, HostListener, Input, OnChanges, OnDestroy, SimpleChanges, ViewChild
} from '@angular/core';
import { HttpErrorResponse } from '@angular/common/http';
import { Observable, Subscription } from 'rxjs';
import { PtEvaluationForm } from './pt-evaluation-form/pt-evaluation-form';
import {
    GAIT_TABS, NO_ACTIONS, PtEvaluationActions, PtEvaluationGeneralResponse, PtEvaluationService,
    PtEvaluationTabKey, PtEvaluationTabResponse, PtEvaluationTabsResponse, PtEvaluationView,
    PtFormResponse, PtFormTarget, PtFormValues, PtReportResponse
} from './pt-evaluation.service';
import { Visit } from '../../visits/visits';

// Button labels from the legacy pages (pt.jsp, pt-eval-*.jsp).
const EDIT_LABELS: Partial<Record<PtEvaluationTabKey, string>> = {
    'prom-strength': 'Edit Data',
    'motor-control': 'Edit Motor Control Values',
    'tone': 'Edit Tone Values',
    'pain': 'Edit Pain Values',
    'hat': 'Edit HAT Scores',
    'dystonia-scale': 'Edit Dystonia Scores',
    // Movement disorder visits only: legacy edits data_pt_movement_disorder.orthoses as "Comments" on this tab.
    'orthotics': 'Edit Comments'
};

export interface PendingOrthosisDelete {
    id: number;
    orthosis: string;
    side: string;
}

type Mode = 'view' | 'form' | 'report';

@Component({
    selector: 'app-pt-evaluation',
    standalone: true,
    imports: [DatePipe, DecimalPipe, NgTemplateOutlet, PtEvaluationForm],
    templateUrl: './pt-evaluation.html',
    styleUrls: ['./pt-evaluation.css']
})
export class PtEvaluation implements OnChanges, OnDestroy {
    @Input({ required: true }) patientId!: number;
    @Input({ required: true }) visitId!: number;
    @ViewChild(PtEvaluationForm) private formComponent?: PtEvaluationForm;

    tabs: PtEvaluationTabsResponse = GAIT_TABS;
    activeTab: PtEvaluationTabKey = 'general';
    mode: Mode = 'view';

    evaluation: PtEvaluationGeneralResponse | null = null;
    tabData: PtEvaluationTabResponse | null = null;
    loading = false;
    errorMessage = '';
    actionError = '';
    busy = false;

    formTarget: PtFormTarget | null = null;
    form: PtFormResponse | null = null;
    saving = false;
    saveError = '';
    serverFieldErrors: Record<string, string> = {};
    conflict = false;

    report: PtReportResponse | null = null;
    pendingDelete: PendingOrthosisDelete | null = null;
    comparisonVisits: Visit[] = [];
    compareVisitIds: (number | null)[] = [null, null];

    private subscription?: Subscription;
    private actionSubscription?: Subscription;
    private comparisonSubscription?: Subscription;

    constructor(
        private readonly service: PtEvaluationService,
        private readonly cdr: ChangeDetectorRef
    ) {}

    get activeLabel(): string {
        return this.tabs.tabs.find(tab => tab.key === this.activeTab)?.label ?? '';
    }

    get isGeneral(): boolean {
        return this.activeTab === 'general';
    }

    get actions(): PtEvaluationActions {
        return (this.isGeneral ? this.evaluation?.actions : this.tabData?.actions) ?? NO_ACTIONS;
    }

    get hasRecord(): boolean {
        return this.isGeneral ? !!this.evaluation?.gmfmD.length : !!this.tabData?.hasData;
    }

    get editLabel(): string {
        if (this.isGeneral || this.activeTab === 'general-md') {
            return this.tabs.movementDisorder ? 'Modify General Data' : 'Modify General or GMFM Data';
        }
        return EDIT_LABELS[this.activeTab] ?? 'Edit Data';
    }

    // Label/value blocks come as tables whose columns are all blank; those render without a header row.
    hasColumnLabels(columns: string[] | null): boolean {
        return !!columns?.some(column => column.trim() !== '');
    }

    hasRowActions(sections: PtEvaluationTabResponse['sections']): boolean {
        return sections.some(section => section.rows?.some(row => row.actions?.edit || row.actions?.delete));
    }

    ngOnChanges(changes: SimpleChanges): void {
        if (changes['patientId'] || changes['visitId']) {
            this.resetEditing();
            this.mode = 'view';
            this.loadTabs();
        }
    }

    ngOnDestroy(): void {
        this.subscription?.unsubscribe();
        this.actionSubscription?.unsubscribe();
        this.comparisonSubscription?.unsubscribe();
    }

    @HostListener('window:beforeunload', ['$event'])
    onBeforeUnload(event: BeforeUnloadEvent): void {
        if (this.hasUnsavedChanges || this.saving) {
            event.preventDefault();
            event.returnValue = '';
        }
    }

    get hasUnsavedChanges(): boolean {
        return this.mode === 'form' && !!this.formComponent?.hasChanges;
    }

    canLeave(): boolean {
        if (this.saving) {
            window.alert('Please wait for the save to finish before leaving.');
            return false;
        }
        return !this.hasUnsavedChanges || window.confirm('Discard unsaved PT evaluation changes?');
    }

    selectTab(key: PtEvaluationTabKey): void {
        if (key === this.activeTab && this.mode === 'view') {
            return;
        }
        if (!this.canLeave()) {
            return;
        }
        this.resetEditing();
        this.mode = 'view';
        this.activeTab = key;
        this.loadView();
    }

    loadTabs(): void {
        if (!this.validIds()) {
            this.subscription?.unsubscribe();
            this.loading = false;
            this.errorMessage = 'Select a valid patient and visit.';
            return;
        }
        this.subscription?.unsubscribe();
        this.loading = true;
        this.errorMessage = '';
        this.subscription = this.service.getTabs(this.patientId, this.visitId).subscribe({
            next: tabs => {
                this.tabs = tabs.tabs.length ? tabs : GAIT_TABS;
                if (!this.tabs.tabs.some(tab => tab.key === this.activeTab)) {
                    this.activeTab = this.tabs.tabs[0].key;
                }
                this.loadView();
            },
            error: error => this.fail(error)
        });
    }

    loadView(): void {
        this.subscription?.unsubscribe();
        this.evaluation = null;
        this.tabData = null;
        this.errorMessage = '';
        this.actionError = '';
        if (!this.validIds()) {
            this.loading = false;
            this.errorMessage = 'Select a valid patient and visit.';
            return;
        }
        this.loading = true;
        const request: Observable<PtEvaluationView> = this.isGeneral
            ? this.service.getGeneral(this.patientId, this.visitId)
            : this.service.getTab(this.patientId, this.visitId, this.activeTab);
        this.subscription = request.subscribe({
            next: view => {
                this.applyView(view);
                this.loading = false;
                this.cdr.markForCheck();
            },
            error: error => this.fail(error)
        });
    }

    createRecord(): void {
        this.runAction(this.service.createRecord(this.patientId, this.visitId, this.activeTab), view => {
            this.applyView(view);
        }, 'Unable to create the record. Please try again.');
    }

    openEdit(): void {
        this.openForm({ tab: this.activeTab });
    }

    openOrthosis(orthosis: number | 'new'): void {
        this.openForm({ tab: 'orthotics', orthosis });
    }

    // Legacy delete_orth.jsp asks on its own page, showing the orthosis and side, before deleting.
    deleteOrthosis(id: number, orthosis: string, side: string | null): void {
        this.actionError = '';
        this.pendingDelete = { id, orthosis, side: side ?? '' };
    }

    confirmDelete(): void {
        if (!this.pendingDelete) {
            return;
        }
        const id = this.pendingDelete.id;
        this.runAction(this.service.deleteOrthosis(this.patientId, this.visitId, id), view => {
            this.pendingDelete = null;
            this.applyView(view);
        }, 'Unable to delete the orthosis. Please try again.');
    }

    cancelDelete(): void {
        this.pendingDelete = null;
        this.actionError = '';
    }

    // Button wording from the legacy modify/insert pages.
    get submitLabel(): string {
        const target = this.formTarget;
        if (target && 'orthosis' in target) {
            return target.orthosis === 'new' ? 'Insert Orthosis' : 'Save Changes';
        }
        return target?.tab === 'general' || target?.tab === 'general-md' ? 'Save All Changes' : 'Save Changes';
    }

    saveForm(values: PtFormValues): void {
        if (!this.form || !this.formTarget || this.saving) {
            return;
        }
        this.saving = true;
        this.saveError = '';
        this.serverFieldErrors = {};
        this.actionSubscription?.unsubscribe();
        this.actionSubscription = this.service.saveForm(this.patientId, this.visitId, this.formTarget, {
            revision: this.form.revision, values
        }).subscribe({
            next: view => {
                this.saving = false;
                this.resetEditing();
                this.mode = 'view';
                this.applyView(view);
                this.cdr.markForCheck();
            },
            error: (error: HttpErrorResponse) => {
                console.error('PT evaluation save failed:', error);
                this.saving = false;
                this.conflict = error.status === 409;
                this.serverFieldErrors = error.status === 400 ? error.error?.fieldErrors ?? {} : {};
                this.saveError = this.conflict
                    ? 'This record was changed by someone else. Reload to see the latest values.'
                    : error.status === 400 ? error.error?.message || 'Please correct the highlighted fields.'
                    : error.status === 403 ? 'You do not have permission to change this record.'
                    : 'Unable to save. Please try again.';
                this.cdr.markForCheck();
            }
        });
    }

    cancelForm(): void {
        if (!this.canLeave()) {
            return;
        }
        this.resetEditing();
        this.mode = 'view';
    }

    reloadForm(): void {
        if (this.formTarget) {
            this.openForm(this.formTarget);
        }
    }

    openReport(): void {
        if (!this.canLeave()) {
            return;
        }
        this.resetEditing();
        this.mode = 'report';
        this.compareVisitIds = [null, null];
        this.comparisonVisits = [];
        this.comparisonSubscription?.unsubscribe();
        // Without other visits the report still works for the current visit alone, so a failure here only hides the picker.
        this.comparisonSubscription = this.service.getComparisonVisits(this.patientId, this.visitId).subscribe({
            next: visits => {
                this.comparisonVisits = visits;
                this.cdr.markForCheck();
            },
            error: error => console.error('Unable to load comparison visits:', error)
        });
        this.loadReport();
    }

    loadReport(): void {
        this.report = null;
        const compareIds = this.compareVisitIds.filter((id): id is number => id !== null);
        this.runAction(this.service.getComparisonReport(this.patientId, this.visitId, compareIds), report => {
            this.report = report;
        }, 'Unable to load the comparison report. Please try again.');
    }

    setCompareVisit(index: 0 | 1, value: string): void {
        const id = value ? Number(value) : null;
        const next = [...this.compareVisitIds];
        next[index] = id;
        if (next[0] === null || next[0] === next[1]) {
            next[1] = null;
        }
        this.compareVisitIds = next;
        this.loadReport();
    }

    comparisonOptions(index: 0 | 1): Visit[] {
        return index === 0
            ? this.comparisonVisits
            : this.comparisonVisits.filter(visit => visit.id !== this.compareVisitIds[0]);
    }

    closeReport(): void {
        this.comparisonSubscription?.unsubscribe();
        this.mode = 'view';
        this.report = null;
        this.comparisonVisits = [];
        this.compareVisitIds = [null, null];
        this.actionError = '';
    }

    print(): void {
        window.print();
    }

    private openForm(target: PtFormTarget): void {
        this.resetEditing();
        this.mode = 'form';
        this.formTarget = target;
        this.runAction(this.service.getForm(this.patientId, this.visitId, target), form => {
            this.form = form;
        }, 'Unable to load the form. Please try again.');
    }

    private runAction<T>(request: Observable<T>, apply: (value: T) => void, message: string): void {
        this.actionSubscription?.unsubscribe();
        this.busy = true;
        this.actionError = '';
        this.actionSubscription = request.subscribe({
            next: value => {
                apply(value);
                this.busy = false;
                this.cdr.markForCheck();
            },
            error: (error: HttpErrorResponse) => {
                console.error('PT evaluation action failed:', error);
                this.busy = false;
                this.actionError = error.status === 403 ? 'You do not have permission to do this.' : message;
                this.cdr.markForCheck();
            }
        });
    }

    private applyView(view: PtEvaluationView): void {
        if ('gmfmD' in view) {
            this.evaluation = view;
            this.tabData = null;
        } else {
            this.tabData = view;
            this.evaluation = null;
        }
    }

    private resetEditing(): void {
        this.actionSubscription?.unsubscribe();
        this.pendingDelete = null;
        this.formTarget = null;
        this.form = null;
        this.saving = false;
        this.busy = false;
        this.saveError = '';
        this.serverFieldErrors = {};
        this.conflict = false;
        this.actionError = '';
    }

    private fail(error: HttpErrorResponse): void {
        console.error('PT evaluation load failed:', error);
        this.errorMessage = 'Unable to load the PT evaluation. Please try again.';
        this.loading = false;
        this.cdr.markForCheck();
    }

    private validIds(): boolean {
        return Number.isInteger(this.patientId) && this.patientId > 0
            && Number.isInteger(this.visitId) && this.visitId > 0;
    }
}












import { provideHttpClient } from '@angular/common/http';
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { vi } from 'vitest';
import { PtEvaluation } from './pt-evaluation';
import {
    PtEvaluationActions, PtEvaluationGeneralResponse, PtEvaluationTabResponse, PtFormField, PtFormResponse,
    PtReportResponse
} from './pt-evaluation.service';

describe('PtEvaluation', () => {
    let fixture: ComponentFixture<PtEvaluation>;
    let http: HttpTestingController;
    const base = '/api/patients/42/visits/7/pt-evaluation';
    const allActions: PtEvaluationActions = { create: true, edit: true, add: true, report: true };

    function general(overrides: Partial<PtEvaluationGeneralResponse> = {}): PtEvaluationGeneralResponse {
        return {
            patientId: 42, visitId: 7,
            general: {
                heightCm: 120.5, heightZ: '(z = 0.42)', weightKg: null, weightZ: null,
                rightFootLengthCm: null, leftFootLengthCm: null, rightFootWidthCm: null, leftFootWidthCm: null,
                ambulationScale: '1 -- Independent ambulation all environments', gmfcs: null,
                fms5m: '6 -- Independent on all surfaces', fms50m: ' -- ', fms500m: ' -- ', comments: null
            },
            gmfmD: [
                { items: '52', task: 'Pull to stand', bilateral: false, right: 0, left: null },
                { items: '54,55', task: 'One leg stance hand held - Right', bilateral: true, right: 0, left: 2 }
            ],
            gmfmModules: [
                { module: 'A', score: null, maxScore: null, percent: null },
                { module: 'D', score: 0, maxScore: 39, percent: 0 }
            ],
            meanPercent: 0,
            actions: allActions,
            ...overrides
        };
    }

    function tab(overrides: Partial<PtEvaluationTabResponse> = {}): PtEvaluationTabResponse {
        return {
            patientId: 42, visitId: 7, tab: 'prom-strength', hasData: true, emptyMessage: null,
            actions: allActions,
            sections: [
                {
                    kind: 'table', title: null,
                    columnGroups: [{ label: '', span: 1 }, { label: 'PROM', span: 2 }, { label: 'Strength', span: 2 }],
                    columns: ['', 'Right', 'Left', 'Right', 'Left'],
                    rows: [
                        { label: 'Hip Flex', cells: ['+120 [N]', 'WNL []', '4+ [N]', null] },
                        { label: '*Clonus', cells: [null, 'present', null, 'absent'], shadedCells: [0, 2] }
                    ],
                    items: null, text: null, footnote: '* Interpretation only'
                },
                {
                    kind: 'table', title: 'Levels', columnGroups: null, columns: ['', ''],
                    rows: [{ label: 'Current Level', cells: ['3'] }], items: null, text: null, footnote: null
                },
                {
                    kind: 'list', title: 'Activities impacted:', columnGroups: null, columns: null, rows: null,
                    items: ['Running', 'Stairs'], text: null, footnote: null
                },
                {
                    kind: 'list', title: 'Aggravating factors:', columnGroups: null, columns: null, rows: null,
                    items: [], text: null, footnote: null
                },
                {
                    kind: 'text', title: 'Comments', columnGroups: null, columns: null, rows: null,
                    items: null, text: 'Tight hamstrings', footnote: null
                }
            ],
            ...overrides
        };
    }

    function orthotics(overrides: Partial<PtEvaluationTabResponse> = {}): PtEvaluationTabResponse {
        return tab({
            tab: 'orthotics',
            sections: [{
                kind: 'table', title: null, columnGroups: null, columns: ['Orthoses', 'Side'],
                rows: [{ label: 'AFO', cells: ['Right'], id: 5, actions: { edit: true, delete: true } }],
                items: null, text: null, footnote: null
            }],
            ...overrides
        });
    }

    function field(name: string, overrides: Partial<PtFormField> = {}): PtFormField {
        return {
            name, label: name, type: 'number', value: null, options: null, required: false,
            min: null, max: null, step: null, maxLength: null, help: null, ...overrides
        };
    }

    function promForm(): PtFormResponse {
        return {
            tab: 'prom-strength', title: 'Edit PROM/Strength', revision: 'r1',
            sections: [
                {
                    title: 'PROM', layout: 'grid', columns: ['Right', 'R Interp'],
                    rows: [{
                        label: 'Hip Flex',
                        fields: [
                            field('hip_flex_r', { label: 'Hip Flex Right', value: '120', min: -90, max: 180 }),
                            field('hip_flex_r_interp', {
                                label: 'Hip Flex Right Interp', type: 'select', value: 'N',
                                options: [{ value: 'N', label: 'N' }, { value: 'D', label: 'D' }]
                            })
                        ]
                    }]
                },
                {
                    title: 'Pain', layout: 'stack', fields: [
                        field('factors', {
                            label: 'Aggravating factors', type: 'multiselect', value: ['1'],
                            options: [{ value: '1', label: 'walking' }, { value: '2', label: 'running' }]
                        }),
                        field('comments', { label: 'Comments', type: 'textarea', value: 'old', maxLength: 10 })
                    ]
                }
            ]
        };
    }

    function page(): HTMLElement {
        return fixture.nativeElement as HTMLElement;
    }

    function text(element: Element | null): string {
        return element?.textContent?.replace(/\s+/g, ' ').trim() ?? '';
    }

    function cells(row: Element): string[] {
        return Array.from(row.querySelectorAll('th, td')).map(text);
    }

    function button(label: string): HTMLButtonElement {
        const match = (Array.from(page().querySelectorAll('button')) as HTMLButtonElement[])
            .find(item => text(item) === label);
        expect(match, `Expected button "${label}"`).toBeDefined();
        return match!;
    }

    function click(label: string): void {
        button(label).click();
        fixture.detectChanges();
    }

    function input(name: string): HTMLInputElement {
        return page().querySelector(`#pt-${name}`) as HTMLInputElement;
    }

    function type(name: string, value: string): void {
        const element = input(name);
        element.value = value;
        element.dispatchEvent(new Event('input'));
        fixture.detectChanges();
    }

    function load(body = general(), tabs: unknown = null): void {
        fixture.componentRef.setInput('patientId', 42);
        fixture.componentRef.setInput('visitId', 7);
        fixture.detectChanges();
        const tabsRequest = http.expectOne(`${base}/tabs`);
        if (tabs) {
            tabsRequest.flush(tabs);
        } else {
            tabsRequest.flush('', { status: 404, statusText: 'Not Found' });
        }
        http.expectOne(`${base}/general`).flush(body);
        fixture.detectChanges();
    }

    function openTab(label: string, url: string, body: PtEvaluationTabResponse): void {
        click(label);
        http.expectOne(url).flush(body);
        fixture.detectChanges();
    }

    function generalRows(): Map<string, string[]> {
        const rows = Array.from(page().querySelectorAll('.general-table tr'));
        return new Map(rows.map(cells).map(row => [row[0], row]));
    }

    beforeEach(async () => {
        await TestBed.configureTestingModule({
            imports: [PtEvaluation],
            providers: [provideHttpClient(), provideHttpClientTesting()]
        }).compileComponents();
        fixture = TestBed.createComponent(PtEvaluation);
        http = TestBed.inject(HttpTestingController);
        vi.spyOn(console, 'error').mockImplementation(() => {});
    });

    afterEach(() => {
        http.verify();
        vi.restoreAllMocks();
    });

    describe('General & GMFM view', () => {
        it('renders the legacy display strings verbatim, including z-scores', () => {
            load();
            const rows = generalRows();
            expect(rows.get('Height (cm)')).toEqual(['Height (cm)', '120.5 (z = 0.42)', 'Weight (kg)', '']);
            expect(rows.get('Ambulation Scale')).toEqual(['Ambulation Scale', '1 -- Independent ambulation all environments']);
            expect(rows.get('FMS (5m)')).toEqual(['FMS (5m)', '6 -- Independent on all surfaces']);
            expect(rows.get('FMS (50m)')).toEqual(['FMS (50m)', '--']);
        });

        it('shows GMFCS only when the backend provides it', () => {
            const body = general();
            body.general.gmfcs = 'I (Calculated) -- Walks without limitations';
            load(body);
            expect(generalRows().get('GMFCS')).toEqual(['GMFCS', 'I (Calculated) -- Walks without limitations']);
        });

        it('renders GMFM-D tasks with Left values only for bilateral items, and module scores', () => {
            load();
            expect(Array.from(page().querySelectorAll('.gmfm-table tbody tr')).map(cells)).toEqual([
                ['Pull to stand (52)', '0', ''],
                ['One leg stance hand held - Right (54,55)', '0', '2']
            ]);
            expect(Array.from(page().querySelectorAll('.module-list li')).map(text))
                .toEqual(['No entry for Module A', 'Module D: 0 / 39 = 0 %']);
            expect(text(page().querySelector('.mean-percentage'))).toBe('Mean Percentage: 0 %');
        });

        it('shows the legacy General button labels and the comparison report button', () => {
            load();
            expect(button('Modify General or GMFM Data')).toBeTruthy();
            expect(button('View Printable Comparison Report')).toBeTruthy();
        });

        it('hides buttons the user is not allowed to use', () => {
            load(general({ actions: { create: false, edit: false, add: false, report: false } }));
            expect(page().querySelector('.toolbar')).toBeNull();
        });
    });

    describe('general editing', () => {
        it('uses the legacy modify_pt.jsp button wording', () => {
            load();
            click('Modify General or GMFM Data');
            http.expectOne(`${base}/general/form`).flush({ ...promForm(), tab: 'general' });
            fixture.detectChanges();
            expect(button('Save All Changes')).toBeTruthy();
            expect(button('Return without Saving Changes')).toBeTruthy();
        });
    });

    describe('empty records', () => {
        it('offers Create New Record when there is no PT record and replaces the view with the result', () => {
            load(general({ gmfmD: [], gmfmModules: [], meanPercent: null }));
            expect(text(page().querySelector('.empty-state p'))).toBe('No general data has been entered yet for this visit.');
            click('Create New Record');
            const request = http.expectOne(`${base}/general/record`);
            expect(request.request.method).toBe('POST');
            request.flush(general());
            fixture.detectChanges();
            expect(page().querySelector('.general-table')).not.toBeNull();
        });

        it('hides Create New Record without the create permission', () => {
            load(general({ gmfmD: [], gmfmModules: [], meanPercent: null, actions: { ...allActions, create: false } }));
            expect(text(page().querySelector('.empty-state'))).not.toContain('Create New Record');
        });

        it('shows the tab empty message, or nothing when legacy shows nothing', () => {
            load();
            openTab('Pain', `${base}/pain`, tab({ tab: 'pain', hasData: false, emptyMessage: 'No pain data entered.', sections: [] }));
            expect(text(page().querySelector('.empty-state p'))).toBe('No pain data entered.');
            openTab('Orthotics', `${base}/orthotics`, tab({ tab: 'orthotics', hasData: false, emptyMessage: null, sections: [],
                actions: { ...allActions, create: false } }));
            expect(page().querySelector('.empty-state p')).toBeNull();
        });
    });

    describe('generic tabs', () => {
        it('renders server-formatted cells, grouped headers, shaded cells and footnotes', () => {
            load();
            openTab('PROM/Strength', `${base}/prom-strength`, tab());
            expect(text(page().querySelector('h2'))).toBe('PT Evaluation: PROM/Strength');
            const table = page().querySelector('.tab-table')!;
            const headerRows = Array.from(table.querySelectorAll('thead tr'));
            expect(headerRows.map(cells)).toEqual([['', 'PROM', 'Strength'], ['', 'Right', 'Left', 'Right', 'Left']]);
            expect(headerRows[0].children[1].getAttribute('colspan')).toBe('2');
            const bodyRows = Array.from(table.querySelectorAll('tbody tr'));
            expect(bodyRows.map(cells)).toEqual([
                ['Hip Flex', '+120 [N]', 'WNL []', '4+ [N]', ''],
                ['*Clonus', '', 'present', '', 'absent']
            ]);
            expect(Array.from(bodyRows[1].querySelectorAll('td')).map(cell => cell.classList.contains('shaded')))
                .toEqual([true, false, true, false]);
            expect(text(page().querySelector('.footnote'))).toBe('* Interpretation only');
            expect(button('Edit Data')).toBeTruthy();
        });

        it('renders label/value tables without headers, lists, empty lists as nothing, and text', () => {
            load();
            openTab('PROM/Strength', `${base}/prom-strength`, tab());
            const levels = page().querySelectorAll('.tab-table')[1];
            expect(levels.querySelector('thead')).toBeNull();
            expect(Array.from(page().querySelectorAll('.item-list li')).map(text)).toEqual(['Running', 'Stairs']);
            expect(page().querySelectorAll('.item-list').length).toBe(1);
            expect(text(page().querySelector('.comments'))).toBe('Tight hamstrings');
        });

        it('uses the legacy edit button label per tab', () => {
            load();
            openTab('Motor Control', `${base}/motor-control`, tab({ tab: 'motor-control' }));
            expect(button('Edit Motor Control Values')).toBeTruthy();
            openTab('Tone', `${base}/tone`, tab({ tab: 'tone' }));
            expect(button('Edit Tone Values')).toBeTruthy();
            openTab('Pain', `${base}/pain`, tab({ tab: 'pain' }));
            expect(button('Edit Pain Values')).toBeTruthy();
        });

        it('reloads the tab list and active tab when the visit changes', () => {
            load();
            openTab('Tone', `${base}/tone`, tab({ tab: 'tone' }));
            fixture.componentRef.setInput('visitId', 8);
            fixture.detectChanges();
            http.expectOne('/api/patients/42/visits/8/pt-evaluation/tabs').flush('', { status: 404, statusText: 'Not Found' });
            http.expectOne('/api/patients/42/visits/8/pt-evaluation/tone').flush(tab({ tab: 'tone', visitId: 8 }));
        });
    });

    describe('editing', () => {
        function openEditor(): void {
            load();
            openTab('PROM/Strength', `${base}/prom-strength`, tab());
            click('Edit Data');
            http.expectOne(`${base}/prom-strength/form`).flush(promForm());
            fixture.detectChanges();
        }

        it('renders grid and stacked fields with their current values', () => {
            openEditor();
            expect(text(page().querySelector('.form-title'))).toBe('Edit PROM/Strength');
            expect(input('hip_flex_r').value).toBe('120');
            expect(input('hip_flex_r').getAttribute('aria-label')).toBe('Hip Flex Right');
            expect((input('hip_flex_r_interp') as unknown as HTMLSelectElement).value).toBe('N');
            const factors = Array.from(page().querySelectorAll('[aria-label="Aggravating factors"] input')) as HTMLInputElement[];
            expect(factors.map(box => box.checked)).toEqual([true, false]);
            expect(input('comments').value).toBe('old');
        });

        it('blocks saving out-of-range values with the field rules', () => {
            openEditor();
            type('hip_flex_r', '500');
            click('Save Changes');
            http.expectNone(`${base}/prom-strength/form`);
            expect(text(page().querySelector('#pt-hip_flex_r-error'))).toBe('Hip Flex Right must be at most 180.');
            expect(input('hip_flex_r').getAttribute('aria-invalid')).toBe('true');
        });

        it('saves with the revision and returns to the updated view', () => {
            openEditor();
            type('hip_flex_r', '110');
            (page().querySelectorAll('[aria-label="Aggravating factors"] input')[1] as HTMLInputElement).click();
            fixture.detectChanges();
            click('Save Changes');
            const request = http.expectOne(`${base}/prom-strength/form`);
            expect(request.request.method).toBe('PUT');
            expect(request.request.body).toEqual({
                revision: 'r1',
                values: { hip_flex_r: '110', hip_flex_r_interp: 'N', factors: ['1', '2'], comments: 'old' }
            });
            request.flush(tab());
            fixture.detectChanges();
            expect(page().querySelector('app-pt-evaluation-form')).toBeNull();
            expect(page().querySelector('.tab-table')).not.toBeNull();
        });

        it('shows server field errors from a 400', () => {
            openEditor();
            click('Save Changes');
            http.expectOne(`${base}/prom-strength/form`).flush(
                { message: 'Please fix the errors.', fieldErrors: { hip_flex_r: 'Must be between -90 and 180' } },
                { status: 400, statusText: 'Bad Request' });
            fixture.detectChanges();
            expect(text(page().querySelector('.form-error'))).toContain('Please fix the errors.');
            expect(text(page().querySelector('#pt-hip_flex_r-error'))).toBe('Must be between -90 and 180');
        });

        it('handles a revision conflict by offering to reload the form', () => {
            openEditor();
            click('Save Changes');
            http.expectOne(`${base}/prom-strength/form`).flush('', { status: 409, statusText: 'Conflict' });
            fixture.detectChanges();
            expect(button('Save Changes').disabled).toBe(true);
            click('Discard changes and reload');
            http.expectOne(`${base}/prom-strength/form`).flush(promForm());
        });

        it('asks before discarding unsaved changes on cancel or tab switch', () => {
            openEditor();
            type('comments', 'new note');
            const confirm = vi.spyOn(window, 'confirm').mockReturnValue(false);
            click('Return without Saving Changes');
            click('Tone');
            expect(confirm).toHaveBeenCalledTimes(2);
            expect(page().querySelector('app-pt-evaluation-form')).not.toBeNull();
            expect(fixture.componentInstance.canLeave()).toBe(false);

            confirm.mockReturnValue(true);
            click('Return without Saving Changes');
            expect(page().querySelector('app-pt-evaluation-form')).toBeNull();
        });
    });

    describe('orthotics', () => {
        it('adds, edits and deletes orthoses through their own endpoints', () => {
            load();
            openTab('Orthotics', `${base}/orthotics`, orthotics());

            click('Add an Orthosis');
            http.expectOne(`${base}/orthotics/new/form`).flush({ ...promForm(), tab: 'orthotics', title: 'Add an Orthosis' });
            fixture.detectChanges();
            click('Insert Orthosis');
            const add = http.expectOne(`${base}/orthotics/new/form`);
            expect(add.request.method).toBe('POST');
            add.flush(orthotics());
            fixture.detectChanges();

            (page().querySelector('[aria-label="Edit AFO"]') as HTMLButtonElement).click();
            fixture.detectChanges();
            http.expectOne(`${base}/orthotics/5/form`).flush({ ...promForm(), tab: 'orthotics' });
            fixture.detectChanges();
            click('Save Changes');
            const edit = http.expectOne(`${base}/orthotics/5/form`);
            expect(edit.request.method).toBe('PUT');
            edit.flush(orthotics());
            fixture.detectChanges();

            (page().querySelector('[aria-label="Delete AFO"]') as HTMLButtonElement).click();
            fixture.detectChanges();
            http.expectNone(`${base}/orthotics/5`);
            const confirmation = page().querySelector('.delete-confirm')!;
            expect(text(confirmation.querySelector('h3'))).toBe('Delete Orthosis');
            expect(Array.from(confirmation.querySelectorAll('td')).map(text)).toEqual(['AFO', 'Right']);
            click('Yes - Delete this Orthosis');
            const remove = http.expectOne(`${base}/orthotics/5`);
            expect(remove.request.method).toBe('DELETE');
            remove.flush(tab({ tab: 'orthotics', hasData: false, emptyMessage: null, sections: [] }));
        });

        it('does not delete when the confirmation is declined', () => {
            load();
            openTab('Orthotics', `${base}/orthotics`, orthotics());
            (page().querySelector('[aria-label="Delete AFO"]') as HTMLButtonElement).click();
            fixture.detectChanges();
            click('Return without Deleting');
            http.expectNone(`${base}/orthotics/5`);
            expect(page().querySelector('.delete-confirm')).toBeNull();
            expect(page().querySelector('[aria-label="Delete AFO"]')).not.toBeNull();
        });

        it('creates and edits the movement disorder comments next to the orthoses list', () => {
            fixture.componentRef.setInput('patientId', 42);
            fixture.componentRef.setInput('visitId', 7);
            fixture.detectChanges();
            http.expectOne(`${base}/tabs`).flush({
                movementDisorder: true,
                tabs: [{ key: 'general-md', label: 'General' }, { key: 'orthotics', label: 'Orthotics' }]
            });
            http.expectOne(`${base}/general-md`).flush(tab({ tab: 'general-md' }));
            fixture.detectChanges();
            openTab('Orthotics', `${base}/orthotics`, orthotics({ actions: { ...allActions, edit: false, create: true } }));
            expect(page().querySelector('[aria-label="Delete AFO"]')).not.toBeNull();
            click('Create New Record');
            const create = http.expectOne(`${base}/orthotics/record`);
            expect(create.request.method).toBe('POST');
            create.flush(orthotics({ actions: { ...allActions, create: false } }));
            fixture.detectChanges();
            expect((Array.from(page().querySelectorAll('button')) as HTMLButtonElement[])
                .some(item => text(item) === 'Create New Record')).toBe(false);
            click('Edit Comments');
            http.expectOne(`${base}/orthotics/form`).flush({ ...promForm(), tab: 'orthotics' });
            fixture.detectChanges();
            expect(button('Save Changes')).toBeTruthy();
            expect(button('Return without Saving Changes')).toBeTruthy();
        });
    });

    describe('comparison report', () => {
        it('loads the report, prints it, and returns to the evaluation', () => {
            load();
            click('View Printable Comparison Report');
            const report: PtReportResponse = {
                title: 'PT Evaluation Comparison', header: [{ label: 'Patient', value: 'Test Patient' }],
                sections: tab().sections
            };
            http.expectOne('/api/patients/42/visits').flush([]);
            http.expectOne(`${base}/comparison-report`).flush(report);
            fixture.detectChanges();
            expect(page().querySelector('.compare-picker')).toBeNull();
            expect(text(page().querySelector('.report-title'))).toBe('PT Evaluation Comparison');
            expect(text(page().querySelector('.report-header dt'))).toBe('Patient');
            expect(text(page().querySelector('.report-header dd'))).toBe('Test Patient');

            const print = vi.spyOn(window, 'print').mockImplementation(() => {});
            click('Print');
            expect(print).toHaveBeenCalled();

            click('Back to PT Evaluation');
            expect(page().querySelector('.general-table')).not.toBeNull();
        });

        it('reloads the report with up to two comparison visits of the same patient', () => {
            load();
            click('View Printable Comparison Report');
            const report: PtReportResponse = { title: 'PT Evaluation Comparison', header: [], sections: [] };
            http.expectOne('/api/patients/42/visits').flush([
                { id: 7, patientId: 42, attendance: 'Yes', date: '2026-09-16', visitType: 'Gait', visitSubType: null },
                { id: 5, patientId: 42, attendance: 'Yes', date: '2025-03-02', visitType: 'Gait', visitSubType: null },
                { id: 3, patientId: 42, attendance: null, date: '2024-01-10', visitType: null, visitSubType: null },
                { id: 2, patientId: 42, attendance: 'No', date: '2023-05-05', visitType: 'Gait', visitSubType: null },
                { id: 9, patientId: 42, attendance: null, date: null, visitType: null, visitSubType: null }
            ]);
            http.expectOne(`${base}/comparison-report`).flush(report);
            fixture.detectChanges();

            const selects = (): HTMLSelectElement[] => Array.from(page().querySelectorAll('.compare-picker select'));
            const optionValues = (select: HTMLSelectElement) => Array.from(select.options).map(option => option.value);
            expect(optionValues(selects()[0])).toEqual(['', '5', '3']);
            expect(selects()[1].disabled).toBe(true);

            selects()[0].value = '5';
            selects()[0].dispatchEvent(new Event('change'));
            http.expectOne(request => request.url === `${base}/comparison-report`
                && request.params.get('compareVisitId1') === '5' && !request.params.has('compareVisitId2')).flush(report);
            fixture.detectChanges();
            expect(selects()[1].disabled).toBe(false);
            expect(optionValues(selects()[1])).toEqual(['', '3']);

            selects()[1].value = '3';
            selects()[1].dispatchEvent(new Event('change'));
            http.expectOne(request => request.params.get('compareVisitId1') === '5'
                && request.params.get('compareVisitId2') === '3').flush(report);
            fixture.detectChanges();

            selects()[0].value = '';
            selects()[0].dispatchEvent(new Event('change'));
            http.expectOne(request => request.url === `${base}/comparison-report` && request.params.keys().length === 0)
                .flush(report);
        });
    });

    describe('movement disorder visits', () => {
        it('uses the tab list from the backend and loads MD tabs generically', () => {
            fixture.componentRef.setInput('patientId', 42);
            fixture.componentRef.setInput('visitId', 7);
            fixture.detectChanges();
            http.expectOne(`${base}/tabs`).flush({
                movementDisorder: true,
                tabs: [
                    { key: 'general-md', label: 'General' }, { key: 'prom-strength', label: 'PROM/Strength' },
                    { key: 'orthotics', label: 'Orthotics' }, { key: 'tone', label: 'Tone' }, { key: 'pain', label: 'Pain' },
                    { key: 'hat', label: 'HAT' }, { key: 'dystonia-scale', label: 'Dystonia Scale' }
                ]
            });
            http.expectOne(`${base}/general-md`).flush(tab({ tab: 'general-md' }));
            fixture.detectChanges();

            expect(Array.from(page().querySelectorAll('.tab')).map(text))
                .toEqual(['General', 'PROM/Strength', 'Orthotics', 'Tone', 'Pain', 'HAT', 'Dystonia Scale']);
            expect(button('Modify General Data')).toBeTruthy();
            openTab('Dystonia Scale', `${base}/dystonia-scale`, tab({ tab: 'dystonia-scale' }));
            expect(text(page().querySelector('h2'))).toBe('PT Evaluation: Dystonia Scale');
        });
    });

    it('shows an error with a retry that reloads', () => {
        fixture.componentRef.setInput('patientId', 42);
        fixture.componentRef.setInput('visitId', 7);
        fixture.detectChanges();
        http.expectOne(`${base}/tabs`).flush('boom', { status: 500, statusText: 'Server Error' });
        fixture.detectChanges();
        expect(text(page().querySelector('[role="alert"]'))).toContain('Unable to load the PT evaluation.');
        click('Try Again');
        http.expectOne(`${base}/tabs`).flush('', { status: 404, statusText: 'Not Found' });
        http.expectOne(`${base}/general`).flush(general());
    });

    it('does not call the API for an invalid visit', () => {
        fixture.componentRef.setInput('patientId', 42);
        fixture.componentRef.setInput('visitId', 0);
        fixture.detectChanges();
        http.expectNone(`${base}/tabs`);
        expect(text(page())).toContain('Select a valid patient and visit.');
    });
});










import { Injectable } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { Observable, catchError, map, of, throwError } from 'rxjs';
import { Visit } from '../../visits/visits';

// The backend ports the legacy clinconn PT pages (pt.jsp, pt-eval-*.jsp, modify_*.jsp) and returns
// display strings already formatted the legacy way, so the page renders them verbatim.

// Which buttons legacy shows for the signed-in user's role; the backend enforces the same rules.
export interface PtEvaluationActions {
    create: boolean;
    edit: boolean;
    add: boolean;
    report: boolean;
}

export const NO_ACTIONS: PtEvaluationActions = { create: false, edit: false, add: false, report: false };

// A visit with no PT record comes back with empty gmfmD/gmfmModules and every general value null.
export interface PtGeneralInfo {
    heightCm: number | null;
    heightZ?: string | null;
    weightKg: number | null;
    weightZ?: string | null;
    rightFootLengthCm: number | null;
    leftFootLengthCm: number | null;
    rightFootWidthCm: number | null;
    leftFootWidthCm: number | null;
    ambulationScale: string | null;
    // Legacy shows GMFCS beside the ambulation scale only for CP diagnoses; null otherwise.
    gmfcs: string | null;
    fms5m: string | null;
    fms50m: string | null;
    fms500m: string | null;
    comments: string | null;
}

export interface GmfmTask {
    items: string;
    task: string;
    bilateral: boolean;
    right: number | null;
    left: number | null;
}

export interface GmfmModuleScore {
    module: 'A' | 'B' | 'C' | 'D' | 'E';
    score: number | null;
    maxScore: number | null;
    percent: number | null;
}

export interface PtEvaluationGeneralResponse {
    patientId: number;
    visitId: number;
    general: PtGeneralInfo;
    gmfmD: GmfmTask[];
    gmfmModules: GmfmModuleScore[];
    meanPercent: number | null;
    actions?: PtEvaluationActions;
}

export type PtEvaluationTabKey =
    | 'general' | 'general-md' | 'prom-strength' | 'orthotics' | 'motor-control' | 'tone' | 'pain'
    | 'hat' | 'dystonia-scale';

export interface PtEvaluationTab {
    key: PtEvaluationTabKey;
    label: string;
}

export interface PtEvaluationTabsResponse {
    movementDisorder: boolean;
    tabs: PtEvaluationTab[];
}

// Used until the backend serves the per-visit tab list; matches pt-tab-header.jsp for gait visits.
export const GAIT_TABS: PtEvaluationTabsResponse = {
    movementDisorder: false,
    tabs: [
        { key: 'general', label: 'General & GMFM' },
        { key: 'prom-strength', label: 'PROM/Strength' },
        { key: 'orthotics', label: 'Orthotics' },
        { key: 'motor-control', label: 'Motor Control' },
        { key: 'tone', label: 'Tone' },
        { key: 'pain', label: 'Pain' }
    ]
};

export interface PtEvaluationTableRow {
    label: string;
    cells: (string | null)[];
    // Cell indexes legacy greys out (e.g. score cells of "*" interpretation-only tests).
    shadedCells?: number[];
    // Present on rows that can be edited or deleted on their own (Orthotics).
    id?: number;
    actions?: { edit: boolean; delete: boolean };
}

export interface PtEvaluationSection {
    kind: 'table' | 'list' | 'text';
    title: string | null;
    columnGroups: { label: string; span: number }[] | null;
    columns: string[] | null;
    rows: PtEvaluationTableRow[] | null;
    items: string[] | null;
    text: string | null;
    footnote: string | null;
}

export interface PtEvaluationTabResponse {
    patientId: number;
    visitId: number;
    tab: PtEvaluationTabKey;
    hasData: boolean;
    emptyMessage: string | null;
    sections: PtEvaluationSection[];
    actions?: PtEvaluationActions;
}

export type PtEvaluationView = PtEvaluationGeneralResponse | PtEvaluationTabResponse;

export interface PtFormOption {
    value: string;
    label: string;
}

export interface PtFormField {
    name: string;
    label: string;
    type: 'number' | 'text' | 'textarea' | 'select' | 'radio' | 'checkbox' | 'multiselect';
    value: string | string[] | null;
    options: PtFormOption[] | null;
    required: boolean;
    min: number | null;
    max: number | null;
    step: number | null;
    maxLength: number | null;
    help: string | null;
}

export interface PtFormSection {
    title: string | null;
    layout: 'grid' | 'stack';
    columns?: string[] | null;
    rows?: { label: string; fields: (PtFormField | null)[] }[] | null;
    fields?: PtFormField[] | null;
}

export interface PtFormResponse {
    tab: PtEvaluationTabKey;
    title: string;
    revision: string;
    sections: PtFormSection[];
}

export type PtFormValues = Record<string, string | string[]>;

export interface PtFormSaveRequest {
    revision: string;
    values: PtFormValues;
}

export interface PtReportResponse {
    title: string;
    header: { label: string; value: string | null }[];
    sections: PtEvaluationSection[];
}

// The orthotics form is per orthosis: "new" for Add an Orthosis, or the orthosis id.
export type PtFormTarget = { tab: PtEvaluationTabKey } | { tab: 'orthotics'; orthosis: number | 'new' };

@Injectable({
    providedIn: 'root'
})
export class PtEvaluationService {
    constructor(private readonly http: HttpClient) {}

    getTabs(patientId: number, visitId: number): Observable<PtEvaluationTabsResponse> {
        return this.http.get<PtEvaluationTabsResponse>(`${this.base(patientId, visitId)}/tabs`).pipe(
            catchError(error => error?.status === 404 ? of(GAIT_TABS) : throwError(() => error))
        );
    }

    getGeneral(patientId: number, visitId: number): Observable<PtEvaluationGeneralResponse> {
        return this.http.get<PtEvaluationGeneralResponse>(`${this.base(patientId, visitId)}/general`);
    }

    getTab(patientId: number, visitId: number, tab: PtEvaluationTabKey): Observable<PtEvaluationTabResponse> {
        return this.http.get<PtEvaluationTabResponse>(`${this.base(patientId, visitId)}/${tab}`);
    }

    createRecord(patientId: number, visitId: number, tab: PtEvaluationTabKey): Observable<PtEvaluationView> {
        return this.http.post<PtEvaluationView>(`${this.base(patientId, visitId)}/${tab}/record`, {});
    }

    getForm(patientId: number, visitId: number, target: PtFormTarget): Observable<PtFormResponse> {
        return this.http.get<PtFormResponse>(this.formUrl(patientId, visitId, target));
    }

    saveForm(
        patientId: number, visitId: number, target: PtFormTarget, request: PtFormSaveRequest
    ): Observable<PtEvaluationView> {
        const url = this.formUrl(patientId, visitId, target);
        return 'orthosis' in target && target.orthosis === 'new'
            ? this.http.post<PtEvaluationView>(url, request)
            : this.http.put<PtEvaluationView>(url, request);
    }

    deleteOrthosis(patientId: number, visitId: number, orthosisId: number): Observable<PtEvaluationTabResponse> {
        return this.http.delete<PtEvaluationTabResponse>(`${this.base(patientId, visitId)}/orthotics/${orthosisId}`);
    }

    // Legacy took up to two comparison visits from the session's "Set Comparison Dates"; here they are passed explicitly.
    getComparisonReport(patientId: number, visitId: number, compareVisitIds: number[] = []): Observable<PtReportResponse> {
        let params = new HttpParams();
        compareVisitIds.slice(0, 2).forEach((id, index) => {
            params = params.set(`compareVisitId${index + 1}`, id);
        });
        return this.http.get<PtReportResponse>(`${this.base(patientId, visitId)}/comparison-report`, { params });
    }

    // Dated, attended visits of the patient other than the current one; legacy's Set Comparison Dates picks by date.
    getComparisonVisits(patientId: number, visitId: number): Observable<Visit[]> {
        return this.http.get<Visit[]>(`/api/patients/${patientId}/visits`).pipe(
            map(visits => visits.filter(visit => {
                const attendance = visit.attendance?.trim().toLowerCase() ?? '';
                return visit.id !== visitId && visit.patientId === patientId && !!visit.date
                    && (attendance === '' || attendance === 'yes');
            }))
        );
    }

    private formUrl(patientId: number, visitId: number, target: PtFormTarget): string {
        return 'orthosis' in target
            ? `${this.base(patientId, visitId)}/orthotics/${target.orthosis}/form`
            : `${this.base(patientId, visitId)}/${target.tab}/form`;
    }

    private base(patientId: number, visitId: number): string {
        return `/api/patients/${patientId}/visits/${visitId}/pt-evaluation`;
    }
}












<section class="pt-evaluation-container" [attr.aria-busy]="loading || busy || saving">
    @if (mode === 'report') {
    <header class="evaluation-header no-print">
        <h2>Comparison Report</h2>
        <div class="evaluation-actions">
            <button type="button" class="primary-button" (click)="print()" [disabled]="!report">Print</button>
            <button type="button" class="secondary-button" (click)="closeReport()">Back to PT Evaluation</button>
        </div>
    </header>
    @if (comparisonVisits.length) {
    <div class="compare-picker no-print">
        @for (index of [0, 1]; track index) {
        <label>
            <span>{{ index === 0 ? 'Compare with' : 'Second comparison' }}</span>
            <select [value]="compareVisitIds[index] ?? ''" [disabled]="busy || (index === 1 && compareVisitIds[0] === null)"
                (change)="setCompareVisit($any(index), $any($event.target).value)">
                <option value="">None</option>
                @for (visit of comparisonOptions($any(index)); track visit.id) {
                <option [value]="visit.id" [selected]="visit.id === compareVisitIds[index]">
                    {{ visit.date | date:'MM/dd/yyyy' }}@if (visit.visitType) { - {{ visit.visitType }}}
                </option>
                }
            </select>
        </label>
        }
    </div>
    }
    @if (busy) {
    <div class="state-message" role="status">Loading comparison report...</div>
    } @else if (actionError) {
    <div class="state-message error" role="alert">
        <p>{{ actionError }}</p>
        <button type="button" (click)="loadReport()">Try Again</button>
    </div>
    } @else if (report) {
    <article class="evaluation-body report">
        <h2 class="report-title">{{ report.title }}</h2>
        @if (report.header.length) {
        <dl class="report-header">
            @for (item of report.header; track $index) {
            <div>
                <dt>{{ item.label }}</dt>
                <dd>{{ item.value ?? '' }}</dd>
            </div>
            }
        </dl>
        }
        <ng-container *ngTemplateOutlet="sectionList; context: { $implicit: report.sections, rowActions: false }" />
    </article>
    }
    } @else {
    <header class="evaluation-header">
        <h2>PT Evaluation: {{ activeLabel }}</h2>
    </header>

    <nav class="evaluation-tabs" aria-label="PT evaluation sections">
        @for (tab of tabs.tabs; track tab.key) {
        <button type="button" class="tab"
            [class.active]="activeTab === tab.key"
            [attr.aria-current]="activeTab === tab.key ? 'page' : null"
            [disabled]="saving"
            (click)="selectTab(tab.key)">{{ tab.label }}</button>
        }
    </nav>

    @if (mode === 'form') {
    @if (busy) {
    <div class="state-message" role="status">Loading form...</div>
    } @else if (actionError) {
    <div class="state-message error" role="alert">
        <p>{{ actionError }}</p>
        <button type="button" (click)="reloadForm()">Try Again</button>
        <button type="button" (click)="cancelForm()">Cancel</button>
    </div>
    } @else if (form) {
    <app-pt-evaluation-form [form]="form" [saving]="saving" [errorMessage]="saveError"
        [serverFieldErrors]="serverFieldErrors" [conflict]="conflict" [submitLabel]="submitLabel"
        (save)="saveForm($event)" (cancel)="cancelForm()" (reload)="reloadForm()" />
    }
    } @else if (loading) {
    <div class="state-message" role="status">Loading PT evaluation...</div>
    } @else if (errorMessage) {
    <div class="state-message error" role="alert">
        <p>{{ errorMessage }}</p>
        <button type="button" (click)="loadTabs()">Try Again</button>
    </div>
    } @else {
    @if (actionError) {
    <p class="action-error" role="alert">{{ actionError }}</p>
    }

    @if (pendingDelete) {
    <section class="evaluation-body delete-confirm" aria-labelledby="pt-delete-title">
        <h3 id="pt-delete-title">Delete Orthosis</h3>
        <p>Are you sure you want to delete this orthosis from this visit?</p>
        <table class="tab-table">
            <tbody>
                <tr><th scope="row">Orthosis:</th><td>{{ pendingDelete.orthosis }}</td></tr>
                <tr><th scope="row">Side:</th><td>{{ pendingDelete.side }}</td></tr>
            </tbody>
        </table>
        <div class="evaluation-actions">
            <button type="button" class="primary-button danger" (click)="confirmDelete()" [disabled]="busy">Yes - Delete this Orthosis</button>
            <button type="button" class="secondary-button" (click)="cancelDelete()" [disabled]="busy">Return without Deleting</button>
        </div>
    </section>
    } @else {
    @if (activeTab === 'orthotics' && actions.add) {
    <div class="toolbar">
        <button type="button" class="secondary-button" (click)="openOrthosis('new')" [disabled]="busy">Add an Orthosis</button>
    </div>
    }

    @if ((evaluation || tabData) && !hasRecord) {
    <div class="empty-state">
        @if (tabData?.emptyMessage || isGeneral) {
        <p>{{ isGeneral ? 'No general data has been entered yet for this visit.' : tabData?.emptyMessage }}</p>
        }
        @if (actions.create) {
        <button type="button" class="primary-button" (click)="createRecord()" [disabled]="busy">Create New Record</button>
        }
    </div>
    } @else if (hasRecord) {
    @if (activeTab === 'orthotics' && actions.create) {
    <div class="toolbar">
        <button type="button" class="primary-button" (click)="createRecord()" [disabled]="busy">Create New Record</button>
    </div>
    }
    @if (actions.edit || actions.report) {
    <div class="toolbar spread">
        @if (actions.edit) {
        <button type="button" class="secondary-button" (click)="openEdit()" [disabled]="busy">{{ editLabel }}</button>
        } @else {
        <span></span>
        }
        @if (actions.report) {
        <button type="button" class="secondary-button" (click)="openReport()" [disabled]="busy">View Printable Comparison Report</button>
        }
    </div>
    }

    @if (tabData) {
    @if (activeTab === 'orthotics' && hasRowActions(tabData.sections)) {
    <p class="hint">Use Edit to change trial information and Delete to remove the trial, device or orthotic.</p>
    }
    <div class="evaluation-body">
        <ng-container *ngTemplateOutlet="sectionList; context: { $implicit: tabData.sections, rowActions: activeTab === 'orthotics' }" />
    </div>
    } @else if (evaluation) {
    <div class="evaluation-body">
        <section class="evaluation-section">
            <h3>General and Marker Information</h3>
            <table class="general-table">
                <tbody>
                    <tr>
                        <th scope="row">Height (cm)</th>
                        <td>{{ evaluation.general.heightCm ?? '' }} {{ evaluation.general.heightZ ?? '' }}</td>
                        <th scope="row">Weight (kg)</th>
                        <td>{{ evaluation.general.weightKg ?? '' }} {{ evaluation.general.weightZ ?? '' }}</td>
                    </tr>
                    <tr>
                        <th scope="row">Right Foot Length (cm)</th>
                        <td>{{ evaluation.general.rightFootLengthCm ?? '' }}</td>
                        <th scope="row">Left Foot Length (cm)</th>
                        <td>{{ evaluation.general.leftFootLengthCm ?? '' }}</td>
                    </tr>
                    <tr>
                        <th scope="row">Right Foot Width (cm)</th>
                        <td>{{ evaluation.general.rightFootWidthCm ?? '' }}</td>
                        <th scope="row">Left Foot Width (cm)</th>
                        <td>{{ evaluation.general.leftFootWidthCm ?? '' }}</td>
                    </tr>
                    <tr>
                        <th scope="row">Ambulation Scale</th>
                        <td colspan="3">{{ evaluation.general.ambulationScale ?? '' }}</td>
                    </tr>
                    @if (evaluation.general.gmfcs) {
                    <tr>
                        <th scope="row">GMFCS</th>
                        <td colspan="3">{{ evaluation.general.gmfcs }}</td>
                    </tr>
                    }
                    <tr>
                        <th scope="row">FMS (5m)</th>
                        <td colspan="3">{{ evaluation.general.fms5m ?? '' }}</td>
                    </tr>
                    <tr>
                        <th scope="row">FMS (50m)</th>
                        <td colspan="3">{{ evaluation.general.fms50m ?? '' }}</td>
                    </tr>
                    <tr>
                        <th scope="row">FMS (500m)</th>
                        <td colspan="3">{{ evaluation.general.fms500m ?? '' }}</td>
                    </tr>
                    <tr>
                        <th scope="row">Comments</th>
                        <td colspan="3" class="comments">{{ evaluation.general.comments ?? '' }}</td>
                    </tr>
                </tbody>
            </table>
        </section>

        <section class="evaluation-section">
            <h3>Functional Strength - GMFM-D</h3>
            <table class="gmfm-table">
                <thead>
                    <tr>
                        <th scope="col">Task</th>
                        <th scope="col" class="score">Right</th>
                        <th scope="col" class="score">Left</th>
                    </tr>
                </thead>
                <tbody>
                    @for (task of evaluation.gmfmD; track task.items) {
                    <tr>
                        <th scope="row">{{ task.task }} ({{ task.items }})</th>
                        <td class="score">{{ task.right ?? '' }}</td>
                        <td class="score">{{ task.bilateral ? (task.left ?? '') : '' }}</td>
                    </tr>
                    }
                </tbody>
            </table>
        </section>

        <section class="evaluation-section">
            <h3>GMFM Modules</h3>
            <ul class="module-list">
                @for (module of evaluation.gmfmModules; track module.module) {
                @if (module.score === null) {
                <li class="no-entry">No entry for Module {{ module.module }}</li>
                } @else {
                <li>Module {{ module.module }}: {{ module.score }} / {{ module.maxScore }}
                    = {{ module.percent | number:'1.0-2' }} %</li>
                }
                }
            </ul>
            <p class="mean-percentage">Mean Percentage:
                {{ evaluation.meanPercent === null ? '--' : (evaluation.meanPercent | number:'1.0-2') + ' %' }}</p>
        </section>
    </div>
    }
    }
    }
    }
    }
</section>

<ng-template #sectionList let-sections let-rowActions="rowActions">
    @for (section of sections; track $index) {
    <section class="evaluation-section">
        @if (section.title) {
        <h3>{{ section.title }}</h3>
        }
        @switch (section.kind) {
        @case ('table') {
        <table class="tab-table">
            @if (section.columnGroups?.length || hasColumnLabels(section.columns)) {
            <thead>
                @if (section.columnGroups?.length) {
                <tr class="column-groups">
                    @for (group of section.columnGroups; track $index) {
                    <th scope="colgroup" [attr.colspan]="group.span">{{ group.label }}</th>
                    }
                </tr>
                }
                @if (hasColumnLabels(section.columns)) {
                <tr>
                    @for (column of section.columns; track $index) {
                    <th scope="col">{{ column }}</th>
                    }
                    @if (rowActions && hasRowActions([section])) {
                    <th scope="col"><span class="visually-hidden">Actions</span></th>
                    }
                </tr>
                }
            </thead>
            }
            <tbody>
                @for (row of section.rows ?? []; track $index) {
                <tr>
                    <th scope="row">{{ row.label }}</th>
                    @for (cell of row.cells; track $index; let cellIndex = $index) {
                    <td [class.shaded]="row.shadedCells?.includes(cellIndex)">{{ cell ?? '' }}</td>
                    }
                    @if (rowActions && hasRowActions([section])) {
                    <td class="row-actions">
                        @if (row.id !== undefined && row.actions?.edit) {
                        <button type="button" class="link-button" (click)="openOrthosis(row.id)" [disabled]="busy"
                            [attr.aria-label]="'Edit ' + row.label">Edit</button>
                        }
                        @if (row.id !== undefined && row.actions?.delete) {
                        <button type="button" class="link-button danger" (click)="deleteOrthosis(row.id, row.label, row.cells[0])"
                            [disabled]="busy" [attr.aria-label]="'Delete ' + row.label">Delete</button>
                        }
                    </td>
                    }
                </tr>
                }
            </tbody>
        </table>
        }
        @case ('list') {
        @if (section.items?.length) {
        <ul class="item-list">
            @for (item of section.items; track $index) {
            <li>{{ item }}</li>
            }
        </ul>
        }
        }
        @case ('text') {
        <p class="comments">{{ section.text ?? '' }}</p>
        }
        }
        @if (section.footnote) {
        <p class="footnote">{{ section.footnote }}</p>
        }
    </section>
    }
</ng-template>


















:host {
    display: block;
    width: 100%;
    min-width: 0;
}

.pt-evaluation-container {
    width: 100%;
    min-width: 0;
    box-sizing: border-box;
    background: #ffffff;
    border: 1px solid #e2e8f0;
    border-radius: 10px;
    overflow: hidden;
}

.evaluation-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 16px;
    padding: 18px 22px;
    border-bottom: 1px solid #e2e8f0;
    flex-wrap: wrap;
}

.evaluation-header h2 {
    margin: 0;
    font-size: 19px;
    font-weight: 650;
    color: #1f2937;
}

.evaluation-actions {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
}

.primary-button,
.secondary-button {
    padding: 9px 18px;
    border: 1px solid #15803d;
    border-radius: 6px;
    background: white;
    color: #15803d;
    font-weight: 600;
    cursor: pointer;
}

.primary-button {
    background: #15803d;
    color: #ffffff;
}

.primary-button:disabled,
.secondary-button:disabled {
    border-color: #cbd5e1;
    background: white;
    color: #94a3b8;
    cursor: not-allowed;
}

.toolbar {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    padding: 14px 22px 0;
}

.toolbar.spread {
    justify-content: space-between;
}

.empty-state {
    padding: 28px 22px;
    font-size: 14px;
    color: #475569;
}

.empty-state p {
    margin: 0 0 12px;
}

.action-error {
    margin: 14px 22px 0;
    padding: 10px 12px;
    border: 1px solid #fecaca;
    border-radius: 6px;
    background: #fef2f2;
    color: #b91c1c;
    font-size: 13px;
}

.hint {
    margin: 12px 22px 0;
    font-size: 13px;
    color: #475569;
}

.shaded {
    background: #e2e8f0;
}

.row-actions {
    white-space: nowrap;
}

.link-button {
    padding: 2px 6px;
    border: none;
    background: none;
    color: #15803d;
    font-weight: 600;
    cursor: pointer;
    text-decoration: underline;
}

.primary-button.danger {
    border-color: #b91c1c;
    background: #b91c1c;
}

.delete-confirm h3 {
    margin: 0 0 8px;
    font-size: 16px;
    color: #1f2937;
}

.delete-confirm p {
    margin: 0 0 12px;
    font-size: 14px;
    color: #334155;
}

.delete-confirm .evaluation-actions {
    margin-top: 14px;
}

.link-button.danger {
    color: #b91c1c;
}

.link-button:disabled {
    color: #94a3b8;
    cursor: not-allowed;
}

.visually-hidden {
    position: absolute;
    width: 1px;
    height: 1px;
    overflow: hidden;
    clip: rect(0 0 0 0);
    white-space: nowrap;
}

.compare-picker {
    display: flex;
    flex-wrap: wrap;
    gap: 12px 20px;
    padding: 14px 22px 0;
}

.compare-picker label {
    display: flex;
    flex-direction: column;
    gap: 4px;
    font-size: 13px;
    font-weight: 600;
    color: #334155;
}

.compare-picker select {
    min-width: 220px;
    padding: 7px 8px;
    border: 1px solid #cbd5e1;
    border-radius: 6px;
    font-size: 13px;
}

.report-title {
    margin: 0 0 12px;
    font-size: 18px;
    color: #1f2937;
}

.report-header {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
    gap: 6px 18px;
    margin: 0 0 18px;
    font-size: 13px;
}

.report-header dt {
    font-weight: 600;
    color: #64748b;
}

.report-header dd {
    margin: 0;
    color: #1f2937;
}

@media print {
    .no-print {
        display: none;
    }

    .pt-evaluation-container {
        border: none;
    }

    .evaluation-section {
        overflow: visible;
        break-inside: avoid;
    }
}

.evaluation-tabs {
    display: flex;
    flex-wrap: wrap;
    gap: 4px;
    padding: 12px 22px 0;
    border-bottom: 1px solid #e2e8f0;
}

.tab {
    padding: 9px 14px;
    border: 1px solid transparent;
    border-bottom: none;
    border-radius: 6px 6px 0 0;
    background: none;
    color: #475569;
    font-weight: 600;
    cursor: pointer;
}

.tab.active {
    border-color: #e2e8f0;
    background: #f0fdf4;
    color: #15803d;
}

.tab-table {
    width: auto;
    min-width: min(100%, 420px);
    max-width: 100%;
}

.tab-table thead th {
    background: #15803d;
    color: #ffffff;
    text-align: center;
}

.tab-table .column-groups th {
    background: #166534;
}

.tab-table tbody th {
    width: auto;
    white-space: nowrap;
}

/* Legacy stacks some cells (e.g. bilateral Dystonia scores, pain factor lists) one value per line. */
.tab-table td {
    white-space: pre;
}

.evaluation-section {
    overflow-x: auto;
}

.item-list {
    margin: 0;
    padding-left: 20px;
    font-size: 13px;
    line-height: 1.7;
    color: #1f2937;
}

.footnote {
    margin: 6px 0 0;
    font-size: 12px;
    color: #64748b;
}

.evaluation-body {
    padding: 18px 22px 24px;
}

.evaluation-section + .evaluation-section {
    margin-top: 26px;
}

.evaluation-section h3 {
    margin: 0 0 10px;
    font-size: 15px;
    font-weight: 650;
    color: #1f2937;
}

table {
    width: 100%;
    border-collapse: collapse;
    font-size: 13px;
}

th,
td {
    padding: 8px 10px;
    border: 1px solid #e2e8f0;
    text-align: left;
    vertical-align: top;
}

tbody th {
    width: 22%;
    background: #f8fafc;
    color: #334155;
    font-weight: 600;
}

.gmfm-table {
    max-width: 560px;
}

.gmfm-table tbody th {
    width: auto;
}

.gmfm-table thead th {
    background: #15803d;
    color: #ffffff;
}

.score {
    width: 70px;
    text-align: center;
}

.comments {
    white-space: pre-wrap;
}

.module-list {
    margin: 0;
    padding: 0;
    list-style: none;
    font-size: 13px;
    line-height: 1.7;
    color: #1f2937;
}

.no-entry {
    color: #64748b;
    font-style: italic;
}

.mean-percentage {
    margin: 10px 0 0;
    padding-top: 10px;
    border-top: 1px solid #e2e8f0;
    font-size: 13px;
    font-weight: 600;
}

.state-message {
    padding: 35px 20px;
    text-align: center;
    color: #475569;
}

.state-message.error {
    color: #b91c1c;
}

.state-message button {
    padding: 8px 14px;
    cursor: pointer;
}

@media (max-width: 640px) {
    .general-table tr {
        display: grid;
        grid-template-columns: 1fr 1fr;
    }

    .general-table th,
    .general-table td {
        width: auto;
    }
}















import { NgTemplateOutlet } from '@angular/common';
import { Component, EventEmitter, Input, OnChanges, Output, SimpleChanges } from '@angular/core';
import { PtFormField, PtFormResponse, PtFormValues } from '../pt-evaluation.service';

// Renders a legacy modify_*.jsp form described by the backend; validation mirrors the field rules it sends.
@Component({
    selector: 'app-pt-evaluation-form',
    standalone: true,
    imports: [NgTemplateOutlet],
    templateUrl: './pt-evaluation-form.html',
    styleUrls: ['./pt-evaluation-form.css']
})
export class PtEvaluationForm implements OnChanges {
    @Input({ required: true }) form!: PtFormResponse;
    @Input() saving = false;
    @Input() errorMessage = '';
    @Input() serverFieldErrors: Record<string, string> = {};
    @Input() conflict = false;
    @Input() submitLabel = 'Save Changes';

    @Output() save = new EventEmitter<PtFormValues>();
    @Output() cancel = new EventEmitter<void>();
    @Output() reload = new EventEmitter<void>();

    values: PtFormValues = {};
    fieldErrors: Record<string, string> = {};
    private initial = '';

    ngOnChanges(changes: SimpleChanges): void {
        if (changes['form']) {
            this.values = {};
            for (const field of this.allFields()) {
                this.values[field.name] = field.type === 'multiselect'
                    ? [...(Array.isArray(field.value) ? field.value : [])]
                    : (Array.isArray(field.value) ? field.value.join(',') : field.value ?? '');
            }
            this.initial = JSON.stringify(this.values);
            this.fieldErrors = {};
        }
        if (changes['serverFieldErrors']) {
            this.fieldErrors = { ...this.serverFieldErrors };
        }
    }

    get hasChanges(): boolean {
        return JSON.stringify(this.values) !== this.initial;
    }

    text(field: PtFormField): string {
        const value = this.values[field.name];
        return Array.isArray(value) ? value.join(',') : value ?? '';
    }

    isChecked(field: PtFormField): boolean {
        const checkedValue = field.options?.[0]?.value ?? 'true';
        return this.values[field.name] === checkedValue;
    }

    isSelected(field: PtFormField, option: string): boolean {
        const value = this.values[field.name];
        return Array.isArray(value) && value.includes(option);
    }

    setValue(field: PtFormField, value: string): void {
        this.values[field.name] = value;
        delete this.fieldErrors[field.name];
    }

    setChecked(field: PtFormField, checked: boolean): void {
        this.setValue(field, checked ? field.options?.[0]?.value ?? 'true' : '');
    }

    toggleOption(field: PtFormField, option: string, selected: boolean): void {
        const current = Array.isArray(this.values[field.name]) ? this.values[field.name] as string[] : [];
        this.values[field.name] = selected
            ? [...current.filter(value => value !== option), option]
            : current.filter(value => value !== option);
        delete this.fieldErrors[field.name];
    }

    submit(): void {
        this.fieldErrors = {};
        for (const field of this.allFields()) {
            const error = this.validate(field);
            if (error) {
                this.fieldErrors[field.name] = error;
            }
        }
        if (Object.keys(this.fieldErrors).length === 0) {
            this.save.emit(structuredClone(this.values));
        }
    }

    private validate(field: PtFormField): string | null {
        const value = this.values[field.name];
        const empty = Array.isArray(value) ? value.length === 0 : (value ?? '').trim() === '';
        if (empty) {
            return field.required ? `${field.label} is required.` : null;
        }
        if (Array.isArray(value)) {
            return null;
        }
        if (field.type === 'number') {
            const number = Number(value);
            if (!Number.isFinite(number)) {
                return `${field.label} must be a number.`;
            }
            if (field.min !== null && number < field.min) {
                return `${field.label} must be at least ${field.min}.`;
            }
            if (field.max !== null && number > field.max) {
                return `${field.label} must be at most ${field.max}.`;
            }
        }
        if (field.maxLength !== null && value.length > field.maxLength) {
            return `${field.label} must be ${field.maxLength} characters or fewer.`;
        }
        return null;
    }

    private allFields(): PtFormField[] {
        return this.form.sections.flatMap(section => [
            ...(section.fields ?? []),
            ...(section.rows ?? []).flatMap(row => row.fields.filter((field): field is PtFormField => field !== null))
        ]);
    }
}
















<form class="pt-form" (submit)="$event.preventDefault(); submit()" novalidate [attr.aria-busy]="saving">
    <h3 class="form-title">{{ form.title }}</h3>

    @if (errorMessage) {
    <div class="form-error" role="alert">
        <p>{{ errorMessage }}</p>
        @if (conflict) {
        <button type="button" class="secondary-button" (click)="reload.emit()">Discard changes and reload</button>
        }
    </div>
    }

    @for (section of form.sections; track $index) {
    <fieldset class="form-section">
        @if (section.title) {
        <legend>{{ section.title }}</legend>
        }
        @if (section.layout === 'grid') {
        <div class="grid-wrapper">
            <table class="form-grid">
                @if (section.columns?.length) {
                <thead>
                    <tr>
                        <th scope="col"></th>
                        @for (column of section.columns; track $index) {
                        <th scope="col">{{ column }}</th>
                        }
                    </tr>
                </thead>
                }
                <tbody>
                    @for (row of section.rows ?? []; track $index) {
                    <tr>
                        <th scope="row">{{ row.label }}</th>
                        @for (field of row.fields; track $index) {
                        <td>
                            @if (field) {
                            <ng-container *ngTemplateOutlet="input; context: { $implicit: field, inGrid: true }" />
                            }
                        </td>
                        }
                    </tr>
                    }
                </tbody>
            </table>
        </div>
        } @else {
        @for (field of section.fields ?? []; track field.name) {
        <div class="stack-field">
            @if (field.type !== 'checkbox') {
            <label class="field-label" [attr.for]="'pt-' + field.name">{{ field.label }}@if (field.required) {<span
                    class="required" aria-hidden="true"> *</span>}</label>
            }
            <ng-container *ngTemplateOutlet="input; context: { $implicit: field, inGrid: false }" />
        </div>
        }
        }
    </fieldset>
    }

    <div class="form-actions">
        <button type="submit" class="primary-button" [disabled]="saving || conflict">{{ saving ? 'Saving...' : submitLabel }}</button>
        <button type="button" class="secondary-button" (click)="cancel.emit()" [disabled]="saving">Return without Saving Changes</button>
    </div>
</form>

<ng-template #input let-field let-inGrid="inGrid">
    @switch (field.type) {
    @case ('textarea') {
    <textarea [id]="'pt-' + field.name" [attr.aria-label]="inGrid ? field.label : null" rows="4"
        [value]="text(field)" [attr.maxlength]="field.maxLength" [disabled]="saving"
        [attr.aria-invalid]="fieldErrors[field.name] ? true : null"
        [attr.aria-describedby]="fieldErrors[field.name] ? 'pt-' + field.name + '-error' : null"
        (input)="setValue(field, $any($event.target).value)"></textarea>
    }
    @case ('select') {
    <select [id]="'pt-' + field.name" [attr.aria-label]="inGrid ? field.label : null" [disabled]="saving"
        [attr.aria-invalid]="fieldErrors[field.name] ? true : null"
        [attr.aria-describedby]="fieldErrors[field.name] ? 'pt-' + field.name + '-error' : null"
        (change)="setValue(field, $any($event.target).value)">
        @if (!field.required) {
        <option value="" [selected]="text(field) === ''"></option>
        }
        @for (option of field.options ?? []; track option.value) {
        <option [value]="option.value" [selected]="text(field) === option.value">{{ option.label }}</option>
        }
    </select>
    }
    @case ('radio') {
    <div class="choice-group" role="radiogroup" [attr.aria-label]="field.label">
        @for (option of field.options ?? []; track option.value) {
        <label class="choice">
            <input type="radio" [name]="'pt-' + field.name" [value]="option.value" [disabled]="saving"
                [checked]="text(field) === option.value" (change)="setValue(field, option.value)" />
            {{ option.label }}
        </label>
        }
    </div>
    }
    @case ('checkbox') {
    <label class="choice">
        <input type="checkbox" [id]="'pt-' + field.name" [disabled]="saving" [checked]="isChecked(field)"
            (change)="setChecked(field, $any($event.target).checked)" />
        {{ field.label }}
    </label>
    }
    @case ('multiselect') {
    <div class="choice-group" role="group" [attr.aria-label]="field.label">
        @for (option of field.options ?? []; track option.value) {
        <label class="choice">
            <input type="checkbox" [value]="option.value" [disabled]="saving"
                [checked]="isSelected(field, option.value)"
                (change)="toggleOption(field, option.value, $any($event.target).checked)" />
            {{ option.label }}
        </label>
        }
    </div>
    }
    @default {
    <input [type]="field.type === 'number' ? 'number' : 'text'" [id]="'pt-' + field.name"
        [attr.aria-label]="inGrid ? field.label : null" [value]="text(field)" [disabled]="saving"
        [attr.min]="field.min" [attr.max]="field.max" [attr.step]="field.step" [attr.maxlength]="field.maxLength"
        [class.grid-input]="inGrid"
        [attr.aria-invalid]="fieldErrors[field.name] ? true : null"
        [attr.aria-describedby]="fieldErrors[field.name] ? 'pt-' + field.name + '-error' : null"
        (input)="setValue(field, $any($event.target).value)" />
    }
    }
    @if (field.help) {
    <span class="field-help">{{ field.help }}</span>
    }
    @if (fieldErrors[field.name]) {
    <span class="field-error" [id]="'pt-' + field.name + '-error'">{{ fieldErrors[field.name] }}</span>
    }
</ng-template>
















:host {
    display: block;
}

.pt-form {
    padding: 18px 22px 24px;
}

.form-title {
    margin: 0 0 14px;
    font-size: 16px;
    font-weight: 650;
    color: #1f2937;
}

.form-section {
    margin: 0 0 18px;
    padding: 12px 14px;
    border: 1px solid #e2e8f0;
    border-radius: 8px;
    min-width: 0;
}

.form-section legend {
    padding: 0 6px;
    font-size: 14px;
    font-weight: 650;
    color: #1f2937;
}

.grid-wrapper {
    overflow-x: auto;
}

.form-grid {
    border-collapse: collapse;
    font-size: 13px;
}

.form-grid th,
.form-grid td {
    padding: 5px 8px;
    border: 1px solid #e2e8f0;
    text-align: left;
    vertical-align: top;
}

.form-grid thead th {
    background: #15803d;
    color: #ffffff;
    text-align: center;
}

.form-grid tbody th {
    background: #f8fafc;
    color: #334155;
    font-weight: 600;
    white-space: nowrap;
}

.stack-field {
    display: flex;
    flex-direction: column;
    gap: 4px;
    margin-bottom: 12px;
    font-size: 13px;
}

.field-label {
    font-weight: 600;
    color: #334155;
}

.required {
    color: #b91c1c;
}

input[type='text'],
input[type='number'],
select,
textarea {
    box-sizing: border-box;
    max-width: 100%;
    padding: 6px 8px;
    border: 1px solid #cbd5e1;
    border-radius: 5px;
    font: inherit;
    font-size: 13px;
}

.grid-input {
    width: 90px;
}

textarea {
    width: 100%;
    resize: vertical;
}

[aria-invalid='true'] {
    border-color: #b91c1c;
}

.choice-group {
    display: flex;
    flex-wrap: wrap;
    gap: 6px 14px;
}

.choice {
    display: inline-flex;
    align-items: center;
    gap: 6px;
}

.field-help {
    display: block;
    font-size: 12px;
    color: #64748b;
}

.field-error {
    display: block;
    font-size: 12px;
    color: #b91c1c;
}

.form-error {
    margin-bottom: 14px;
    padding: 10px 12px;
    border: 1px solid #fecaca;
    border-radius: 6px;
    background: #fef2f2;
    color: #b91c1c;
    font-size: 13px;
}

.form-error p {
    margin: 0 0 6px;
}

.form-actions {
    display: flex;
    gap: 8px;
}

.primary-button,
.secondary-button {
    padding: 9px 18px;
    border: 1px solid #15803d;
    border-radius: 6px;
    background: white;
    color: #15803d;
    font-weight: 600;
    cursor: pointer;
}

.primary-button {
    background: #15803d;
    color: #ffffff;
}

.primary-button:disabled,
.secondary-button:disabled {
    opacity: 0.6;
    cursor: not-allowed;
}

















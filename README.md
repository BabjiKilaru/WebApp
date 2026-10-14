import { TestBed } from '@angular/core/testing';
import { ActivatedRouteSnapshot, Router, RouterStateSnapshot, provideRouter } from '@angular/router';
import { of, firstValueFrom, Observable } from 'rxjs';
import { vi } from 'vitest';
import { AuthService } from './auth.service';
import { adminGuard, authGuard } from './auth.guard';

describe('Route access guards', () => {
  const auth = { check: vi.fn(), isAdmin: vi.fn() };
  let router: Router;
  beforeEach(() => {
    vi.clearAllMocks();
    TestBed.configureTestingModule({
      providers: [provideRouter([]), { provide: AuthService, useValue: auth }],
    });
    router = TestBed.inject(Router);
  });
  async function check(guard: typeof authGuard | typeof adminGuard, url: string): Promise<string | boolean> {
    const state = { url } as RouterStateSnapshot;
    const route = {} as ActivatedRouteSnapshot;
    const result = TestBed.runInInjectionContext(() => guard(route, state));
    const resolved = await firstValueFrom(result as Observable<boolean | ReturnType<Router['createUrlTree']>>);
    return typeof resolved === 'boolean' ? resolved : router.serializeUrl(resolved);
  }
  it('allows an authenticated user', async () => {
    auth.check.mockReturnValue(of(true));
    expect(await check(authGuard, '/patients')).toBe(true);
  });
  it('redirects signed-out users to login, preserving their destination', async () => {
    auth.check.mockReturnValue(of(false));
    expect(await check(authGuard, '/patients?tab=history')).toBe('/login?returnUrl=%2Fpatients%3Ftab%3Dhistory');
  });
  it('allows administrators into /admin', async () => {
    auth.check.mockReturnValue(of(true));
    auth.isAdmin.mockReturnValue(true);
    expect(await check(adminGuard, '/admin')).toBe(true);
  });
  it('redirects non-admin authenticated users to patients', async () => {
    auth.check.mockReturnValue(of(true));
    auth.isAdmin.mockReturnValue(false);
    expect(await check(adminGuard, '/admin')).toBe('/patients');
  });
  it('redirects signed-out visitors to login for admin routes', async () => {
    auth.check.mockReturnValue(of(false));
    expect(await check(adminGuard, '/admin')).toBe('/login?returnUrl=%2Fadmin');
  });
});










import { provideHttpClient } from '@angular/common/http';
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';
import { TestBed } from '@angular/core/testing';
import { AuthService } from './auth.service';

const USER = { userid: 'doctor', access: 'admin', firstName: 'Jane', lastName: 'Doe' };

describe('AuthService backend session flow', () => {
  let service: AuthService;
  let http: HttpTestingController;
  beforeEach(() => {
    TestBed.configureTestingModule({ providers: [provideHttpClient(), provideHttpClientTesting()] });
    service = TestBed.inject(AuthService);
    http = TestBed.inject(HttpTestingController);
  });
  afterEach(() => http.verify());
  it('stores the authenticated user only after a successful login response', () => {
    service.login('doctor', 'secret').subscribe();
    expect(service.user()).toBeNull();
    const request = http.expectOne('/api/auth/login');
    expect(request.request.method).toBe('POST');
    expect(request.request.body).toEqual({ userid: 'doctor', password: 'secret' });
    request.flush(USER);
    expect(service.user()).toEqual(USER);
    expect(service.isAdmin()).toBe(true);
  });
  it('recognizes an expired session, clears user and returns false', () => {
    service.user.set(USER);
    let authenticated = true;
    service.check().subscribe(result => authenticated = result);
    http.expectOne('/api/auth/me').flush({}, { status: 401, statusText: 'Unauthorized' });
    expect(authenticated).toBe(false);
    expect(service.user()).toBeNull();
  });
  it('revalidates a session and updates the current user', () => {
    let authenticated = false;
    service.check().subscribe(result => authenticated = result);
    http.expectOne('/api/auth/me').flush(USER);
    expect(authenticated).toBe(true);
    expect(service.user()).toEqual(USER);
  });
  it('clears local state even if logout fails at the server', () => {
    service.user.set(USER);
    service.logout().subscribe();
    const request = http.expectOne('/api/auth/logout');
    expect(request.request.method).toBe('POST');
    request.flush(null, { status: 503, statusText: 'Service unavailable' });
    expect(service.user()).toBeNull();
  });
});









import { provideHttpClient } from "@angular/common/http";
import { provideHttpClientTesting } from "@angular/common/http/testing";
import { TestBed } from "@angular/core/testing";

import { AuthService } from "./auth.service";

describe("AuthService initials", () => {
  let auth: AuthService;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [provideHttpClient(), provideHttpClientTesting()],
    });
    auth = TestBed.inject(AuthService);
  });

  it("uses the uppercase first and last initials", () => {
    auth.user.set({
      userid: "jane.doe",
      access: "user",
      firstName: "jane",
      lastName: "doe",
    });

    expect(auth.initials()).toBe("JD");
  });

  it("uses a single initial when either name is missing", () => {
    auth.user.set({
      userid: "jane.doe",
      access: "user",
      firstName: "jane",
      lastName: null,
    });

    expect(auth.initials()).toBe("J");
  });

  it("falls back to one userid initial when both names are missing", () => {
    auth.user.set({
      userid: "jane.doe",
      access: "user",
      firstName: null,
      lastName: null,
    });

    expect(auth.initials()).toBe("J");
  });
});









import { TestBed } from '@angular/core/testing';
import { provideHttpClient } from '@angular/common/http';
import {
  HttpTestingController,
  provideHttpClientTesting
} from '@angular/common/http/testing';

import { HealthService } from './health';

describe('HealthService', () => {
  let service: HealthService;
  let httpTesting: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        provideHttpClient(),
        provideHttpClientTesting()
      ]
    });

    service = TestBed.inject(HealthService);
    httpTesting = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpTesting.verify();
  });

  it('should retrieve the backend health message as text', () => {
    let result: string | undefined;

    service.getHealth().subscribe(message => {
      result = message;
    });

    const request = httpTesting.expectOne('/api/health');

    expect(request.request.method).toBe('GET');
    expect(request.request.responseType).toBe('text');

    request.flush('Gait Lab backend is running');

    expect(result).toBe('Gait Lab backend is running');
  });
});









import { provideHttpClient } from '@angular/common/http';
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';
import { TestBed } from '@angular/core/testing';
import { Forms } from './forms';

const PATIENT = {
  visitId: 10, patientId: 1, mrn: '123', firstName: 'Sam', lastName: 'Doe',
  visitType: 'Gait', visitSubType: 'Follow-up', visitDate: '2026-10-13',
  appointmentTime: '09:00', recommendedForms: ['patientform', 'invalid', 'patientform', 'analysisform'],
};

describe('Forms', () => {
  let component: Forms;
  let http: HttpTestingController;
  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [Forms], providers: [provideHttpClient(), provideHttpClientTesting()],
    }).overrideComponent(Forms, { set: { template: '' } }).compileComponents();
    component = TestBed.createComponent(Forms).componentInstance;
    http = TestBed.inject(HttpTestingController);
  });
  afterEach(() => http.verify());

  it('builds six weeks of calendar dates', () => {
    component.buildCalendar();
    expect(component.calendarDays).toHaveLength(42);
    expect(component.calendarYears).toHaveLength(101);
  });
  it('does not change months while generating a PDF', () => {
    const month = component.calendarMonth.getMonth();
    component.generatingPdf = true;
    component.changeMonth(1);
    expect(component.calendarMonth.getMonth()).toBe(month);
  });
  it('loads scheduled patients and filters unsupported/duplicate form codes', () => {
    component.selectedDate = '2026-10-13';
    component.loadPatients();
    const req = http.expectOne(request => request.url === '/api/forms/patients');
    expect(req.request.params.get('date')).toBe('2026-10-13');
    req.flush([PATIENT]);
    expect(component.patients[0].selectedForms).toEqual(['patientform', 'analysisform']);
    expect(component.patients[0].selected).toBe(false);
    expect(component.loadedDate).toBe('2026-10-13');
    expect(component.loadingPatients).toBe(false);
  });
  it('shows an error if patient listing fails', () => {
    component.loadPatients();
    http.expectOne(request => request.url === '/api/forms/patients').flush(null, { status: 500, statusText: 'Internal error' });
    expect(component.errorMessage).toContain('Unable to load');
    expect(component.patients).toEqual([]);
    expect(component.loadingPatients).toBe(false);
  });
  it('requires at least one selected patient before generating forms', () => {
    component.viewPrintableForms();
    expect(component.errorMessage).toBe('select at least one patient.');
    http.expectNone('/api/forms/preview');
  });
  it('requires forms for every selected patient', () => {
    component.patients = [{ ...PATIENT, recommendedForms: [], selectedForms: [], selected: true }];
    component.viewPrintableForms();
    expect(component.errorMessage).toBe('Select at least one form for each selected patient.');
    http.expectNone('/api/forms/preview');
  });
  it('respects the manual/automatic form-selection mode', () => {
    component.patients = [{ ...PATIENT, recommendedForms: ['analysisform'], selectedForms: [], selected: true }];
    component.onAutomaticSelectionChange();
    expect(component.patients[0].selectedForms).toEqual(['analysisform']);
    const checkbox = { target: { checked: true } } as unknown as Event;
    component.onFormSelectionChange(component.patients[0], 'ptueform', checkbox);
    expect(component.patients[0].selectedForms).not.toContain('ptueform');
    component.chooseAutomatically = false;
    component.onFormSelectionChange(component.patients[0], 'ptueform', checkbox);
    expect(component.patients[0].selectedForms).toContain('ptueform');
  });
  it('POSTs the selected visit IDs and handles PDF-generation failure', () => {
    component.patients = [{ ...PATIENT, recommendedForms: ['patientform'], selectedForms: ['patientform'], selected: true }];
    component.viewPrintableForms();
    const req = http.expectOne('/api/forms/preview');
    expect(req.request.method).toBe('POST');
    expect(req.request.body).toEqual({ visits: [{ visitId: 10, forms: ['patientform'] }] });
    req.flush(null, { status: 500, statusText: 'Internal error' });
    expect(component.generatingPdf).toBe(false);
    expect(component.errorMessage).toContain('Unable to generate');
  });
});









import { TestBed } from '@angular/core/testing';
import { Router, provideRouter } from '@angular/router';
import { of } from 'rxjs';
import { vi } from 'vitest';
import { AuthService } from '../core/auth.service';
import { Header } from './header';

describe('Header', () => {
  const auth = { user: vi.fn(), check: vi.fn(), logout: vi.fn() };
  let component: Header;
  let router: Router;
  beforeEach(async () => {
    vi.clearAllMocks();
    auth.user.mockReturnValue({ userid: 'clinician' });
    auth.check.mockReturnValue(of(true));
    auth.logout.mockReturnValue(of(null));
    await TestBed.configureTestingModule({
      imports: [Header],
      providers: [provideRouter([]), { provide: AuthService, useValue: auth }],
    }).overrideComponent(Header, { set: { template: '' } }).compileComponents();
    component = TestBed.createComponent(Header).componentInstance;
    router = TestBed.inject(Router);
  });
  it('refreshes user session only when not already loaded', () => {
    component.ngOnInit();
    expect(auth.check).not.toHaveBeenCalled();
    auth.user.mockReturnValue(null);
    component.ngOnInit();
    expect(auth.check).toHaveBeenCalledOnce();
  });
  it('toggles the navigation and user menu, and closes on outside clicks', () => {
    component.toggleMenu();
    expect(component.menuOpen).toBe(true);
    component.closeMenu();
    expect(component.menuOpen).toBe(false);
    const event = { stopPropagation: vi.fn() } as unknown as Event;
    component.toggleUserMenu(event);
    expect(event.stopPropagation).toHaveBeenCalledOnce();
    expect(component.userMenuOpen).toBe(true);
    component.onDocumentClick();
    expect(component.userMenuOpen).toBe(false);
  });
  it('logs out and returns to login', () => {
    const navigate = vi.spyOn(router, 'navigate').mockResolvedValue(true);
    component.menuOpen = true;
    component.userMenuOpen = true;
    component.logout();
    expect(auth.logout).toHaveBeenCalledOnce();
    expect(navigate).toHaveBeenCalledWith(['/login']);
    expect(component.menuOpen).toBe(false);
    expect(component.userMenuOpen).toBe(false);
  });
});













import { provideHttpClient } from "@angular/common/http";
import { HttpTestingController, provideHttpClientTesting } from "@angular/common/http/testing";
import { ComponentFixture, TestBed } from "@angular/core/testing";
import { provideRouter } from "@angular/router";

import { Login } from "./login";

describe("Login", () => {
  let component: Login;
  let fixture: ComponentFixture<Login>;
  let httpTesting: HttpTestingController;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [Login],
      providers: [
        provideRouter([]),
        provideHttpClient(),
        provideHttpClientTesting(),
      ],
    }).compileComponents();

    httpTesting = TestBed.inject(HttpTestingController);
    fixture = TestBed.createComponent(Login);
    httpTesting.expectOne("/api/health").flush("ok");
    component = fixture.componentInstance;
    await fixture.whenStable();
  });

  afterEach(() => httpTesting.verify());

  it("should create", () => {
    expect(component).toBeTruthy();
  });
});










import { provideHttpClient } from '@angular/common/http';
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';
import { TestBed } from '@angular/core/testing';
import { Router, provideRouter } from '@angular/router';
import { vi } from 'vitest';
import { Login } from './login';

describe('Login credentials and server responses', () => {
  let component: Login;
  let http: HttpTestingController;
  let router: Router;
  beforeEach(async () => {
    await TestBed.configureTestingModule({ imports: [Login], providers: [provideRouter([]), provideHttpClient(), provideHttpClientTesting()] })
      .overrideComponent(Login, { set: { template: '' } }).compileComponents();
    component = TestBed.createComponent(Login).componentInstance;
    http = TestBed.inject(HttpTestingController);
    router = TestBed.inject(Router);
    http.expectOne('/api/health').flush('ok');
  });
  afterEach(() => http.verify());
  it('blocks blank usernames and passwords', () => {
    component.onSubmit();
    expect(component.error()).toContain('username');
    component.username.set('clinician');
    component.onSubmit();
    expect(component.error()).toContain('password');
    http.expectNone('/api/auth/login');
  });
  it('prevents duplicate login submissions', () => {
    component.username.set('clinician');
    component.password.set('P@ssw0rd');
    component.onSubmit();
    component.onSubmit();
    const requests = http.match('/api/auth/login');
    expect(requests).toHaveLength(1);
    requests[0].flush({}, { status: 401, statusText: 'Unauthorized' });
    expect(component.error()).toBe('Invalid username or password.');
  });
  it('navigates after successful login', () => {
    const navigate = vi.spyOn(router, 'navigateByUrl').mockResolvedValue(true);
    component.username.set(' clinician ');
    component.password.set('P@ssw0rd');
    component.onSubmit();
    const request = http.expectOne('/api/auth/login');
    expect(request.request.body.userid).toBe('clinician');
    request.flush({ userid: 'clinician', access: 'user', firstName: 'Test', lastName: 'User' });
    expect(navigate).toHaveBeenCalledWith('/patients');
  });
  it('preserves significant leading and trailing whitespace in passwords (known failing regression)', () => {
    component.username.set('clinician');
    component.password.set(' space-sensitive ');
    component.onSubmit();
    const request = http.expectOne('/api/auth/login');
    expect(request.request.body.password).toBe(' space-sensitive ');
    request.flush({}, { status: 401, statusText: 'Unauthorized' });
  });
});










import { provideHttpClient } from '@angular/common/http';
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';
import { TestBed } from '@angular/core/testing';
import { History } from './history';

describe('Patient History', () => {
  let component: History;
  let http: HttpTestingController;
  beforeEach(async () => {
    await TestBed.configureTestingModule({ imports: [History], providers: [provideHttpClient(), provideHttpClientTesting()] })
      .overrideComponent(History, { set: { template: '' } }).compileComponents();
    component = TestBed.createComponent(History).componentInstance;
    http = TestBed.inject(HttpTestingController);
  });
  afterEach(() => { component.ngOnDestroy(); http.verify(); });
  it('does not load data without a patient', () => {
    component.loadHistory();
    expect(component.loading).toBe(false);
    http.expectNone(request => request.url.includes('/history'));
  });
  it('loads and separates the patient history sections', () => {
    component.patientId = 42;
    component.loadHistory();
    const req = http.expectOne('/api/patients/42/history');
    req.flush({ surgeries: [{ id: 1 }], botox: [], healthHistory: [], healthConditions: [{ id: 2 }] });
    expect(component.surgeries).toHaveLength(1);
    expect(component.healthConditions).toHaveLength(1);
    expect(component.loading).toBe(false);
  });
  it('handles missing optional history sections', () => {
    component.patientId = 42;
    component.loadHistory();
    http.expectOne('/api/patients/42/history').flush({});
    expect(component.surgeries).toEqual([]);
    expect(component.botox).toEqual([]);
  });
  it('handles an HTTP failure without leaving a loading spinner', () => {
    component.patientId = 42;
    component.loadHistory();
    http.expectOne('/api/patients/42/history').flush(null, { status: 500, statusText: 'Error' });
    expect(component.loading).toBe(false);
    expect(component.errorMessage).not.toBe('');
  });
  it('formats sides and missing patient demographics safely', () => {
    expect(component.sideLabel('L')).toBe('Left');
    expect(component.sideLabel('R')).toBe('Right');
    expect(component.patientName).toBe('Not recorded');
    expect(component.patientAge).toBe('Not recorded');
  });
});












import { ComponentFixture, TestBed } from '@angular/core/testing';

import { NewPatient } from './new-patient';

describe('NewPatient', () => {

  let component: NewPatient;
  let fixture: ComponentFixture<NewPatient>;

  beforeEach(async () => {

    await TestBed.configureTestingModule({
      imports: [NewPatient]
    })
    .compileComponents();

    fixture = TestBed.createComponent(NewPatient);
    component = fixture.componentInstance;

    fixture.detectChanges();
  });

  it('should create', () => {
    expect(component).toBeTruthy();
  });

  it('rejects missing required patient fields without emitting', () => {
    const added: unknown[] = [];
    component.add.subscribe(patient => added.push(patient));
    component.addPatient();
    expect(component.errorMessage).toBe('First Name is required.');
    component.form.firstName = 'Sam';
    component.addPatient();
    expect(component.errorMessage).toBe('Last Name is required.');
    component.form.lastName = 'Jones';
    component.addPatient();
    expect(component.errorMessage).toBe('MRN is required.');
    component.form.mrn = '123';
    component.addPatient();
    expect(component.errorMessage).toBe('Date of Birth is required.');
    expect(added).toEqual([]);
  });

  it('trims identifying fields and emits the requested record', () => {
    component.form = { firstName: ' Sam ', lastName: ' Jones ', mrn: ' 123 ', nickname: '', gender: '', generalDiagnosisCode: null, dateOfBirth: '2012-04-01', comment: ' Patient note ' };
    const added: unknown[] = [];
    component.add.subscribe(patient => added.push(patient));
    component.addPatient();
    expect(added).toHaveLength(1);
    expect(added[0]).toMatchObject({ firstName: 'Sam', lastName: 'Jones', mrn: '123', comment: 'Patient note' });
  });

  it('selects a diagnosis and closes the dropdown', () => {
    component.diagnosisDropdownOpen = true;
    component.selectDiagnosis({ id: 1, code: 'G01', diagnosis: 'Example' });
    expect(component.form.generalDiagnosisCode).toBe(1);
    expect(component.selectedDiagnosisLabel).toContain('Example');
    expect(component.diagnosisDropdownOpen).toBe(false);
  });
});










import { ComponentFixture, TestBed } from '@angular/core/testing';
import { provideRouter } from '@angular/router';
import { of, throwError } from 'rxjs';
import { afterEach, beforeEach, describe, expect, it, vi } from 'vitest';

import { PatientSearch } from './patient-search';
import { PatientService } from '../patient.service';
import { Patient } from '../patient';

describe('PatientSearch', () => {
  let component: PatientSearch;
  let fixture: ComponentFixture<PatientSearch>;
  let testPatient: Patient;

  const patientServiceMock = {
    getGeneralDiagnoses: vi.fn(),
    searchPatients: vi.fn(),
    createPatient: vi.fn(),
    updatePatient: vi.fn()
  };

  beforeEach(async () => {
    vi.resetAllMocks();

    patientServiceMock.getGeneralDiagnoses.mockReturnValue(of([]));
    patientServiceMock.searchPatients.mockReturnValue(of([]));

    await TestBed.configureTestingModule({
      imports: [PatientSearch],
      providers: [
        provideRouter([]),
        {
          provide: PatientService,
          useValue: patientServiceMock
        }
      ]
    }).compileComponents();

    fixture = TestBed.createComponent(PatientSearch);
    component = fixture.componentInstance;

    testPatient = {
      ...component.patient,
      id: 1125,
      mrn: '123456',
      firstName: 'John',
      lastName: 'Smith',
      dateOfBirth: '2010-05-20',
      gender: 'Male'
    };

    fixture.detectChanges();
  });

  afterEach(() => {
    vi.clearAllTimers();
    vi.useRealTimers();
  });

  it('should create', () => {
    expect(component).toBeTruthy();
  });

  it('should not search with fewer than two characters', () => {
    component.onSearchInput('S');

    expect(patientServiceMock.searchPatients).not.toHaveBeenCalled();
    expect(component.loading).toBe(false);
  });

  it('should return patient suggestions after the delay', () => {
    vi.useFakeTimers();
    patientServiceMock.searchPatients.mockReturnValue(of([testPatient]));

    component.onSearchInput('Smi');

    expect(patientServiceMock.searchPatients).not.toHaveBeenCalled();

    vi.advanceTimersByTime(200);

    expect(patientServiceMock.searchPatients).toHaveBeenCalledWith('Smi');
    expect(component.suggestions).toEqual([testPatient]);
    expect(component.loading).toBe(false);
  });

  it('should search the same name again after clearing', () => {
    vi.useFakeTimers();
    patientServiceMock.searchPatients.mockReturnValue(of([testPatient]));

    component.onSearchInput('Smith');
    vi.advanceTimersByTime(200);

    component.clearPatient();

    component.onSearchInput('Smith');
    vi.advanceTimersByTime(200);

    expect(patientServiceMock.searchPatients).toHaveBeenCalledTimes(2);
    expect(component.suggestions).toEqual([testPatient]);
    expect(component.loading).toBe(false);
  });

  it('should handle a trailing space without getting stuck', () => {
    vi.useFakeTimers();
    patientServiceMock.searchPatients.mockReturnValue(of([testPatient]));

    component.onSearchInput('Smith');
    vi.advanceTimersByTime(200);

    component.onSearchInput('Smith ');
    vi.advanceTimersByTime(200);

    expect(component.suggestions).toEqual([testPatient]);
    expect(component.loading).toBe(false);
  });

  it('should select a patient', () => {
    component.selectPatient(testPatient);

    expect(component.patient).toEqual(testPatient);
    expect(component.patientFound).toBe(true);
    expect(component.searchQuery).toContain('Smith');
  });

  it('should show Visit Home after opening a visit', () => {
    const visit = {
      id: 5,
      patientId: testPatient.id,
      attendance: 'Yes',
      date: '2025-12-03',
      visitType: 'Gait',
      visitSubType: 'Step Watch'
    };

    component.selectPatient(testPatient);
    component.openVisitHome(visit);

    expect(component.selectedVisit).toEqual(visit);
    expect(component.activeTab).toBe('visit-home');
  });

  it('should clear the patient', () => {
    component.selectPatient(testPatient);
    component.openVisitHome({
      id: 5,
      patientId: testPatient.id,
      attendance: 'Yes',
      date: '2025-12-03',
      visitType: 'Gait',
      visitSubType: 'Step Watch'
    });
    component.clearPatient();

    expect(component.patientFound).toBe(false);
    expect(component.searchQuery).toBe('');
    expect(component.patient.id).toBe(0);
    expect(component.selectedVisit).toBeNull();
  });

  it('should change tabs', () => {
    component.selectTab('history');

    expect(component.activeTab).toBe('history');
  });

  it('should select the Visit Home and Videos tabs', () => {
    component.selectTab('visit-home');
    expect(component.activeTab).toBe('visit-home');

    component.selectTab('videos');
    expect(component.activeTab).toBe('videos');
  });

  it('should handle search errors', () => {
    vi.useFakeTimers();
    patientServiceMock.searchPatients.mockReturnValue(
      throwError(() => new Error('Database failure'))
    );

    component.onSearchInput('Smith');
    vi.advanceTimersByTime(200);

    expect(component.errorMessage).toBe('Unable to search patients.');
    expect(component.loading).toBe(false);
    expect(component.suggestions).toEqual([]);
  });
});











import { provideHttpClient } from '@angular/common/http';
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';
import { TestBed } from '@angular/core/testing';
import { QuestionnaireResponses } from './quest-responses';

describe('QuestionnaireResponses', () => {
  let component: QuestionnaireResponses;
  let http: HttpTestingController;
  beforeEach(async () => {
    await TestBed.configureTestingModule({ imports: [QuestionnaireResponses], providers: [provideHttpClient(), provideHttpClientTesting()] })
      .overrideComponent(QuestionnaireResponses, { set: { template: '' } }).compileComponents();
    component = TestBed.createComponent(QuestionnaireResponses).componentInstance;
    http = TestBed.inject(HttpTestingController);
  });
  afterEach(() => { component.ngOnDestroy(); http.verify(); });
  it('starts without an editable visit or patient record', () => {
    expect(component.editorReady).toBe(false);
    expect(component.visitChoices).toEqual([{ value: null, label: 'Select date' }]);
    expect(component.patientName).toBe('Not recorded');
    expect(component.patientAge).toBe('Not recorded');
  });
  it('loads questionnaire page and dropdown options for the selected patient', () => {
    component.patient = { id: 42, firstName: 'Alex', lastName: 'Smith', dateOfBirth: '2014-01-01', gender: 'F' } as any;
    component.ngOnChanges();
    const page = http.expectOne('/api/patients/42/questionnaire-responses');
    const opts = http.expectOne('/api/patients/42/questionnaire-responses/options');
    page.flush({ baseline: { fields: {}, lists: {}, version: 'a' }, visits: [{ id: 55, date: '2026-10-13', visitType: 'Gait', visitSubType: null }] });
    opts.flush({ relationships: [], premature: [], birthStays: [], ages: [], learning: [], walkingSupport: [], walkingChanges: [], fms: [], medications: [], devices: [], gaitConcerns: [], pain: [] });
    expect(component.patientName).toBe('Smith, Alex');
    expect(component.patientSex).toBe('Female');
    expect(component.visitChoices.some(choice => choice.value === 55)).toBe(true);
    expect(component.loading).toBe(false);
  });
  it('reports a load error when the response API fails', () => {
    component.patient = { id: 42 } as any;
    component.ngOnChanges();
    http.expectOne('/api/patients/42/questionnaire-responses/options').flush([]);
    http.expectOne('/api/patients/42/questionnaire-responses').flush(null, { status: 500, statusText: 'Error' });
    expect(component.pageError).not.toBe('');
  });
  it('closes dropdowns on outside clicks', () => {
    component.openDropdown = 'relationship';
    component.closeDropdowns();
    expect(component.openDropdown).toBeNull();
  });
});







import { ComponentFixture, TestBed } from "@angular/core/testing";
import { provideHttpClient } from "@angular/common/http";
import { HttpTestingController, provideHttpClientTesting } from "@angular/common/http/testing";
import { vi } from "vitest";
import { Visits } from './visits';

describe(
    'Visits',
    () => {
        let component: Visits;
        let fixture: ComponentFixture<Visits>;
        let httpTesting: HttpTestingController;

        beforeEach(
            async () => {
                await TestBed .configureTestingModule({
                    imports:[
                        Visits
                    ],

                    providers: [
                        provideHttpClient(),
                        provideHttpClientTesting()
                    ]
                })
                .compileComponents();

                fixture = TestBed.createComponent(Visits);

                component = fixture.componentInstance;

                httpTesting = TestBed.inject(HttpTestingController);
            }
        );

        afterEach(
            () => {
                httpTesting.verify();
            }
        );

        it(
            'should create',
            () => {
                expect(component) .toBeTruthy();
            }
        );

        it('should default attendance filter to Yes / Blank', () => {
            expect(component.attendanceFilter).toBe('attended');
        });

        it('should open a visit when activated', () => {
            const visit = {
                id: 5,
                patientId: 1125,
                date: '2025-12-03',
                attendance: 'Yes',
                visitType: 'Gait',
                visitSubType: 'Step Watch'
            };
            const visitOpened = vi.spyOn(component.visitOpened, 'emit');

            component.openVisit(visit);

            expect(component.selectedVisitId).toBe(5);
            expect(visitOpened).toHaveBeenCalledWith(visit);
        });

        it('should reset attendance filter to Yes / Blank when clearing filters', () => {
            component.attendanceFilter = 'cancelled';
            component.dateFrom = '2025-01-01';
            component.dateTo = '2025-12-31';

            component.clearFilters();

            expect(component.attendanceFilter).toBe('attended');
            expect(component.dateFrom).toBe('');
            expect(component.dateTo).toBe('');
        });

        it(
            'should load visits for patients',
            () => {
                component.patientId = 1125;

                component.loadVisits();

                const request = httpTesting.expectOne(
                    '/api/patients/1125/visits'
                );

                expect(request.request.method) .toBe('GET');

                request.flush([
                    {
                        id:1,
                        patientId: 1125,
                        date: '2025-12-03',
                        visitType: 'Gait',
                        visitSubType: 'Step Watch'
                    },
                    {
                        id:2,
                        patientId: 1125,
                        date: '2025-11-20',
                        visitType: 'Foot',
                        visitSubType: 'Foot Evaluation'
                    },
                ]);

                expect(component.visits.length) .toBe(2);

                expect(component.visits[0] .visitType) .toBe('Gait');

                expect(component.loading) .toBeFalsy();

            }
        );

        it(
            'should handle backend error',
            () => {
                component.patientId = 1125;

                component.loadVisits();

                const request = httpTesting.expectOne('/api/patients/1125/visits');

                request.flush(
                    'Backend error',
                    {
                        status: 500,
                        statusText: 'Server Error'
                    }
                );

                expect(component.visits.length) .toBe(0);

                expect(component.errorMessage) .toBe('Unable to load visits.');

                expect(component.loading) .toBe(false);

            }
        );

        it('should not call backend when patientId is null', () => {
            component.patientId = null;
            component.loadVisits();
            expect(component.visits.length) .toBe(0);

            httpTesting.expectNone(() => true);
        });

        it('separates attended, canceled, no-show and rescheduled visits', () => {
            component.allVisits = [
              { id: 1, patientId: 42, attendance: 'Yes', date: '2026-10-01', visitType: null, visitSubType: null },
              { id: 2, patientId: 42, attendance: 'No Show', date: '2026-10-02', visitType: null, visitSubType: null },
              { id: 3, patientId: 42, attendance: 'Rescheduled', date: '2026-10-03', visitType: null, visitSubType: null },
              { id: 4, patientId: 42, attendance: null, date: '2026-10-04', visitType: null, visitSubType: null },
            ];
            component.applyFilters();
            expect(component.visits.map(visit => visit.id)).toEqual([1, 4]);
            component.attendanceFilter = 'cancelled';
            component.applyFilters();
            expect(component.visits.map(visit => visit.id)).toEqual([2]);
            component.attendanceFilter = 'rescheduled';
            component.applyFilters();
            expect(component.visits.map(visit => visit.id)).toEqual([3]);
            component.attendanceFilter = 'attended';
            component.dateFrom = '2026-10-02';
            component.applyFilters();
            expect(component.visits.map(visit => visit.id)).toEqual([4]);
        });
    }
);











import { provideHttpClient } from '@angular/common/http';
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';
import { TestBed } from '@angular/core/testing';
import { PatientService } from './patient.service';

describe('PatientService HTTP contract', () => {
  let service: PatientService;
  let http: HttpTestingController;
  beforeEach(() => {
    TestBed.configureTestingModule({ providers: [provideHttpClient(), provideHttpClientTesting()] });
    service = TestBed.inject(PatientService);
    http = TestBed.inject(HttpTestingController);
  });
  afterEach(() => http.verify());

  it('gets a patient by ID and a patient by encoded MRN', () => {
    service.getPatientById(42).subscribe();
    service.getByMrn('A B/123').subscribe();
    expect(http.expectOne('/api/patients/42').request.method).toBe('GET');
    expect(http.expectOne('/api/patients/mrn/A%20B%2F123').request.method).toBe('GET');
  });
  it('uses query parameters for search rather than interpolating unsanitized URLs', () => {
    service.searchPatients('Smith & Jones').subscribe();
    const req = http.expectOne(request => request.url === '/api/patients/search');
    expect(req.request.params.get('query')).toBe('Smith & Jones');
    req.flush([]);
    service.searchByLastName("O'Neil").subscribe();
    const lastNameReq = http.expectOne(request => request.url === '/api/patients');
    expect(lastNameReq.request.params.get('lastName')).toBe("O'Neil");
    lastNameReq.flush([]);
  });
  it('sends create and update payloads using POST and PUT', () => {
    const patient = { firstName: 'Avery', mrn: '100' };
    service.createPatient(patient).subscribe();
    service.updatePatient(42, patient).subscribe();
    const create = http.expectOne('/api/patients');
    expect(create.request.method).toBe('POST');
    expect(create.request.body).toEqual(patient);
    create.flush({});
    const update = http.expectOne('/api/patients/42');
    expect(update.request.method).toBe('PUT');
    expect(update.request.body).toEqual(patient);
    update.flush({});
  });
  it('retrieves diagnosis reference data', () => {
    service.getGeneralDiagnoses().subscribe();
    const request = http.expectOne('/api/patients/general-diagnoses');
    expect(request.request.method).toBe('GET');
    request.flush([]);
  });
});







import { provideHttpClient } from '@angular/common/http';
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { provideRouter } from '@angular/router';

import { AuthService } from '../core/auth.service';
import { Profile } from './profile';

describe('Profile password requirements', () => {
    let component: Profile;
    let fixture: ComponentFixture<Profile>;
    let http: HttpTestingController;

    beforeEach(async () => {
        await TestBed.configureTestingModule({
            imports: [Profile],
            providers: [provideRouter([]), provideHttpClient(), provideHttpClientTesting()],
        }).compileComponents();
        TestBed.inject(AuthService).user.set({
            userid: 'test', access: 'user', firstName: 'Test', lastName: 'User',
        });
        http = TestBed.inject(HttpTestingController);
        fixture = TestBed.createComponent(Profile);
        component = fixture.componentInstance;
        fixture.detectChanges();
        http.expectOne('/api/profile').flush({ userid: 'test' });
        component.openPasswordDialog();
        fixture.detectChanges();
        await fixture.whenStable();
    });

    afterEach(() => http.verify());

    it('shows five unmet requirements directly below confirmation initially', () => {
        const list: HTMLUListElement = fixture.nativeElement.querySelector('#passwordRequirements');
        expect(list.children.length).toBe(5);
        expect(list.querySelectorAll('.passed').length).toBe(0);
        expect(list.previousElementSibling?.querySelector('#confirmPassword')).not.toBeNull();
        expect(list.textContent).toContain('Not met:');
        expect(list.textContent).toContain('\u2717');
    });

    it('updates each check as the user types and rechecks confirmation on either field change', async () => {
        const newPassword: HTMLInputElement = fixture.nativeElement.querySelector('#newPassword');
        const confirmation: HTMLInputElement = fixture.nativeElement.querySelector('#confirmPassword');
        const type = async (input: HTMLInputElement, value: string) => {
            input.value = value;
            input.dispatchEvent(new Event('input'));
            await fixture.whenStable();
            fixture.detectChanges();
        };

        await type(newPassword, 'Abcdef1!');
        expect(fixture.nativeElement.querySelectorAll('.password-requirements .passed').length).toBe(4);
        await type(confirmation, 'Abcdef1!');
        expect(fixture.nativeElement.querySelectorAll('.password-requirements .passed').length).toBe(5);
        expect(fixture.nativeElement.querySelector('#passwordRequirements').textContent).toContain('\u2713');
        await type(newPassword, 'Abcdef2!');
        expect(component.passwordRequirements[4].passed).toBe(false);
        await type(confirmation, '');
        await type(newPassword, '');
        expect(component.passwordRequirements.every(requirement => !requirement.passed)).toBe(true);
    });

    it.each([
        ['Abcd1!', 0],
        ['Abcdefg!', 1],
        ['abcdef1!', 2],
        ['Abcdef12', 3],
        ['Abcdef1?', 3],
    ])('rejects %s with its failed requirement and does not submit', (password, index) => {
        component.passwordForm = { currentPassword: 'Oldpass1!', newPassword: password, confirmPassword: password };
        component.changePassword();
        expect(component.passwordRequirements[index].passed).toBe(false);
        expect(component.passwordError()).toBe(component.passwordRequirements[index].label);
        http.expectNone('/api/profile/password');
    });

    it('rejects missing fields and mismatched confirmation', () => {
        component.changePassword();
        expect(component.passwordError()).toBe('All password fields are required.');
        component.passwordForm = {
            currentPassword: 'Oldpass1!', newPassword: 'Abcdef1!', confirmPassword: 'Abcdef2!',
        };
        component.changePassword();
        expect(component.passwordError()).toBe(component.passwordRequirements[4].label);
        http.expectNone('/api/profile/password');
    });

    it.each('!@#$%^&*'.split(''))('accepts %s as a special character and submits once', special => {
        component.passwordForm = {
            currentPassword: 'Oldpass1!', newPassword: `Abcdef1${special}`, confirmPassword: `Abcdef1${special}`,
        };
        component.changePassword();
        component.changePassword();
        const request = http.expectOne('/api/profile/password');
        expect(request.request.method).toBe('PUT');
        expect(request.request.body).toEqual(component.passwordForm);
        request.flush(null);
        expect(component.passwordDialogOpen()).toBe(false);
        expect(component.successMessage()).toBe('Your password was changed successfully.');
    });

    it('preserves the maximum length and different-password restrictions', () => {
        const tooLong = 'Abcdef1!' + 'a'.repeat(121);
        component.passwordForm = {
            currentPassword: 'Oldpass1!', newPassword: tooLong, confirmPassword: tooLong,
        };
        component.changePassword();
        expect(component.passwordError()).toBe('New password must be 128 characters or fewer.');
        component.passwordForm = {
            currentPassword: 'Abcdef1!', newPassword: 'Abcdef1!', confirmPassword: 'Abcdef1!',
        };
        component.changePassword();
        expect(component.passwordError()).toBe('New password must be different from the current password.');
        http.expectNone('/api/profile/password');
    });

    it('resets the checklist when the dialog is reopened', () => {
        component.passwordForm.newPassword = 'Abcdef1!';
        component.passwordForm.confirmPassword = 'Abcdef1!';
        component.closePasswordDialog();
        component.openPasswordDialog();
        expect(component.passwordRequirements.every(requirement => !requirement.passed)).toBe(true);
    });
});









import { vi } from 'vitest';
import { provideHttpClient } from '@angular/common/http';
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';
import { TestBed } from '@angular/core/testing';
import { FirstVisitQuestionnaire } from './first-visit-questionnaire';

describe('FirstVisitQuestionnaire', () => {
  let component: FirstVisitQuestionnaire;
  let http: HttpTestingController;
  beforeEach(async () => {
    await TestBed.configureTestingModule({ imports: [FirstVisitQuestionnaire], providers: [provideHttpClient(), provideHttpClientTesting()] })
      .overrideComponent(FirstVisitQuestionnaire, { set: { template: '' } }).compileComponents();
    component = TestBed.createComponent(FirstVisitQuestionnaire).componentInstance;
    http = TestBed.inject(HttpTestingController);
  });
  afterEach(() => http.verify());
  it('tracks seven required baseline answers', () => {
    component.section = { fields: { historyPerson: 'Parent', historyRelationship: '--', premature: null, birthStay: null, ageWalk: null, ageTalk: null, learning: null }, lists: {}, version: '1' };
    expect(component.getProgress()).toEqual({ answered: 1, total: 7 });
    expect(component.getInvalidFields()).toContain('historyRelationship');
    expect(component.validate()).not.toBeNull();
  });
  it('accepts complete answers including numeric learning codes', () => {
    component.section = { fields: { historyPerson: 'Parent', historyRelationship: 'Mother', premature: 'No', birthStay: 'None', ageWalk: '12m', ageTalk: '24m', learning: 0 }, lists: {}, version: '1' };
    expect(component.getProgress()).toEqual({ answered: 7, total: 7 });
    expect(component.validate()).toBeNull();
  });
  it('does not write a baseline without a patient ID', () => {
    const saved = vi.fn();
    component.saveComplete.subscribe(saved);
    component.save();
    expect(saved).toHaveBeenCalledOnce();
    http.expectNone(request => request.method === 'POST' || request.method === 'PUT');
  });
});












import { provideHttpClient } from '@angular/common/http';
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';
import { TestBed } from '@angular/core/testing';
import { HipQuestionnaire } from './hip-questionnaire';

describe('HipQuestionnaire', () => {
  let component: HipQuestionnaire;
  let http: HttpTestingController;
  beforeEach(async () => {
    await TestBed.configureTestingModule({ imports: [HipQuestionnaire], providers: [provideHttpClient(), provideHttpClientTesting()] })
      .overrideComponent(HipQuestionnaire, { set: { template: '' } }).compileComponents();
    component = TestBed.createComponent(HipQuestionnaire).componentInstance;
    http = TestBed.inject(HttpTestingController);
  });
  afterEach(() => http.verify());
  it('starts with missing WOMAC, UCLA and Harris answers', () => {
    expect(component.getProgress().total).toBeGreaterThan(0);
    expect(component.getProgress().answered).toBe(0);
    expect(component.validate()).not.toBeNull();
  });
  it('recognizes all required answers, including zero scores', () => {
    for (const item of [...component.womacPain, ...component.womacStiffness, ...component.womacFunction]) component.womac[item.key] = 0;
    for (const item of component.harrisItems) component.harris[item.key] = 0;
    component.ucla = 0;
    expect(component.getInvalidFields()).toEqual([]);
    expect(component.getProgress().answered).toBe(component.getProgress().total);
    expect(component.validate()).toBeNull();
  });
  it('POSTs the visit-associated HIP answers and handles save errors', () => {
    component.patientId = 42;
    component.visitId = 10;
    component.save();
    const request = http.expectOne('/api/patients/42/hip-questionnaire');
    expect(request.request.method).toBe('POST');
    expect(request.request.body.visitId).toBe(10);
    request.flush(null, { status: 500, statusText: 'Error' });
    expect(component.saving).toBe(false);
    expect(component.saved).toBe(false);
  });
});









import { vi } from 'vitest';
import { provideHttpClient } from '@angular/common/http';
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';
import { TestBed } from '@angular/core/testing';
import { HistoryQuestionnaire } from './history-questionnaire';

describe('HistoryQuestionnaire', () => {
  let component: HistoryQuestionnaire;
  let http: HttpTestingController;
  beforeEach(async () => {
    await TestBed.configureTestingModule({ imports: [HistoryQuestionnaire], providers: [provideHttpClient(), provideHttpClientTesting()] })
      .overrideComponent(HistoryQuestionnaire, { set: { template: '' } }).compileComponents();
    component = TestBed.createComponent(HistoryQuestionnaire).componentInstance;
    http = TestBed.inject(HttpTestingController);
  });
  afterEach(() => http.verify());
  it('requires the reporter and gait details when those sections are enabled', () => {
    component.items = { fields: {}, lists: {}, version: '1' };
    expect(component.getProgress().total).toBe(7);
    expect(component.getInvalidFields()).toContain('reportingPerson');
    expect(component.getInvalidFields()).toContain('walkingSupportCode');
    expect(component.validate()).not.toBeNull();
  });
  it('excludes hidden sections from validation and progress', () => {
    component.items = { fields: {}, lists: {}, version: '1' };
    component.showWho = false;
    component.showGait = false;
    expect(component.getProgress()).toEqual({ answered: 0, total: 0 });
    expect(component.getInvalidFields()).toEqual([]);
    expect(component.validate()).toBeNull();
  });
  it('does not send a save request without a valid loaded visit', () => {
    const saved = vi.fn();
    component.saveComplete.subscribe(saved);
    component.save();
    expect(saved).toHaveBeenCalledOnce();
    http.expectNone(request => request.method === 'POST' || request.method === 'PUT');
  });
});










import { PODCI_BANK } from './podci-bank';

describe('PODCI question bank consistency', () => {
  it('defines CH, AP, and AS variants', () => {
    expect(Object.keys(PODCI_BANK).sort()).toEqual(['AP', 'AS', 'CH']);
  });
  it('has nonempty, sequential page numbers within each variant', () => {
    for (const [variant, pages] of Object.entries(PODCI_BANK)) {
      expect(pages.length, `${variant} pages`).toBeGreaterThan(0);
      const numbers = pages.map(page => page.n);
      expect(new Set(numbers).size, `${variant} page numbers`).toBe(numbers.length);
      expect(pages.every(page => Array.isArray(page.blocks)), `${variant} blocks`).toBe(true);
    }
  });
});










import { vi } from 'vitest';
import { provideHttpClient } from '@angular/common/http';
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';
import { TestBed } from '@angular/core/testing';
import { PodciQuestionnaire } from './podci-questionnaire';

describe('PODCIQuestionnaire', () => {
  let component: PodciQuestionnaire;
  let http: HttpTestingController;
  beforeEach(async () => {
    await TestBed.configureTestingModule({ imports: [PodciQuestionnaire], providers: [provideHttpClient(), provideHttpClientTesting()] })
      .overrideComponent(PodciQuestionnaire, { set: { template: '' } }).compileComponents();
    component = TestBed.createComponent(PodciQuestionnaire).componentInstance;
    http = TestBed.inject(HttpTestingController);
  });
  afterEach(() => http.verify());
  it('uses the child parent-report variant and calculates unanswered questions', () => {
    expect(component.variant).toBe('CH');
    expect(component.pages.length).toBeGreaterThan(0);
    expect(component.getProgress().total).toBeGreaterThan(0);
    expect(component.getInvalidFields().length).toBeGreaterThan(0);
  });
  it('records radio responses and notifies progress subscribers', () => {
    const progress = vi.fn();
    component.progressChange.subscribe(progress);
    component.setRadio('q2_061', 4);
    expect(component.selected('q2_061', 4)).toBe(true);
    expect(progress).toHaveBeenCalledOnce();
  });
  it('clears dependent grid values when the disabling answer is selected', () => {
    component.answers['q2_023b'] = 1;
    component.answers['q2_023c'] = 2;
    component.setGrid('q2_023a', 2);
    expect(component.gridDisabled('q2_023b')).toBe(true);
    expect(component.answers['q2_023b']).toBeNull();
    expect(component.answers['q2_023c']).toBeNull();
  });
  it('saves only to the selected patient and visit', () => {
    component.patientId = 42;
    component.visitId = 10;
    component.save();
    const request = http.expectOne('/api/patients/42/podci-questionnaire');
    expect(request.request.method).toBe('POST');
    expect(request.request.body.visitId).toBe(10);
    expect(request.request.body.variant).toBe('ch');
    request.flush({});
    expect(component.saved).toBe(true);
  });
});









import { vi } from 'vitest';
import { QuestionnaireClose } from './questionnaire-close';

describe('QuestionnaireClose', () => {
  it('emits continue and save-and-close when not saving', () => {
    const component = new QuestionnaireClose();
    const continueSpy = vi.fn();
    const closeSpy = vi.fn();
    component.continueQuestionnaire.subscribe(continueSpy);
    component.saveAndClose.subscribe(closeSpy);
    component.continue();
    component.close();
    expect(continueSpy).toHaveBeenCalledOnce();
    expect(closeSpy).toHaveBeenCalledOnce();
  });
  it('blocks actions during an in-flight save', () => {
    const component = new QuestionnaireClose();
    const closeSpy = vi.fn();
    component.saveAndClose.subscribe(closeSpy);
    component.saving = true;
    component.close();
    expect(closeSpy).not.toHaveBeenCalled();
  });
});








import { vi } from 'vitest';
import { QuestionnaireEnd } from './questionnaire-end';

describe('QuestionnaireEnd', () => {
  it('emits a close-window event', () => {
    const component = new QuestionnaireEnd();
    const close = vi.fn();
    component.closeWindow.subscribe(close);
    component.close();
    expect(close).toHaveBeenCalledOnce();
  });
});









import { QuestionnairePatientInfo } from './questionnaire-patient-info';

describe('QuestionnairePatientInfo', () => {
  it.each([['M','Male'], ['Male','Male'], ['F','Female'], ['female','Female'], ['', '-'], ['Other', 'Other']])(
    'displays gender %s as %s', (value, expected) => {
      const component = new QuestionnairePatientInfo();
      component.sex = value;
      expect(component.displaySex).toBe(expected);
    },
  );
});








import { QuestionnaireProgress } from "./questionnaire-progress";

describe("QuestionnaireProgress", () => {
  it("marks only the current questionnaire segment red while incomplete", () => {
    const component = new QuestionnaireProgress();
    component.questionnaires = [
      {
        key: "first",
        name: "First",
        answered: 4,
        total: 4,
        completed: true,
      },
      {
        key: "second",
        name: "Second",
        answered: 2,
        total: 3,
        completed: false,
      },
      {
        key: "third",
        name: "Third",
        answered: 0,
        total: 2,
        completed: false,
      },
    ];
    component.currentKey = "second";

    expect(component.circleBackground).toBe(
      "conic-gradient(#009688 0 33%, #d32f2f 33% 67%, #e5e9ed 67% 100%)",
    );
  });

  it("turns only the current questionnaire segment red again when an answer is cleared", () => {
    const component = new QuestionnaireProgress();
    const current = {
      key: "second",
      name: "Second",
      answered: 3,
      total: 3,
      completed: true,
    };
    component.questionnaires = [
      {
        key: "first",
        name: "First",
        answered: 4,
        total: 4,
        completed: true,
      },
      current,
      {
        key: "third",
        name: "Third",
        answered: 0,
        total: 2,
        completed: false,
      },
    ];
    component.currentKey = "second";

    expect(component.circleBackground).toContain("#009688 33% 67%");

    current.answered = 2;

    expect(component.circleBackground).toContain("#d32f2f 33% 67%");
  });


  it('returns zero progress and an empty circle when nothing is selected', () => {
    const component = new QuestionnaireProgress();
    expect(component.current).toBeNull();
    expect(component.currentNumber).toBe(0);
    expect(component.percentage).toBe(0);
    expect(component.circleBackground).toContain('#e5e9ed');
  });

  it('caps the current questionnaire percentage and marks completed segments green', () => {
    const component = new QuestionnaireProgress();
    component.questionnaires = [{ key: 'a', name: 'A', answered: 2, total: 2, completed: true }];
    component.currentKey = 'a';
    expect(component.percentage).toBe(100);
    expect(component.circleBackground).toContain('#009688');
  });
});










import { provideHttpClient } from "@angular/common/http";
import {
  HttpTestingController,
  provideHttpClientTesting,
} from "@angular/common/http/testing";
import { ComponentFixture, TestBed } from "@angular/core/testing";
import {
  ActivatedRoute,
  convertToParamMap,
  provideRouter,
} from "@angular/router";

import { QuestionnaireSession } from "./questionnaire-session";

describe("QuestionnaireSession", () => {
  let component: QuestionnaireSession;
  let fixture: ComponentFixture<QuestionnaireSession>;
  let httpTesting: HttpTestingController;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [QuestionnaireSession],
      providers: [
        provideRouter([]),
        provideHttpClient(),
        provideHttpClientTesting(),
        {
          provide: ActivatedRoute,
          useValue: {
            snapshot: {
              queryParamMap: convertToParamMap({
                patientId: "1",
                visitId: "10",
                selections: "podci-parent",
                language: "en",
              }),
            },
          },
        },
      ],
    }).compileComponents();

    httpTesting = TestBed.inject(HttpTestingController);
    fixture = TestBed.createComponent(QuestionnaireSession);
    component = fixture.componentInstance;
    fixture.detectChanges();

    httpTesting.expectOne("/api/patients/1").flush({});
    httpTesting
      .expectOne("/api/patients/1/questionnaire-picker/assemble")
      .flush([{ code: "PODCI_CH", visitId: 10 }]);
  });

  afterEach(() => httpTesting.verify());

  it("shows the child parent-reported label on the welcome page", () => {
    expect(component.questionnaireNames).toEqual([
      "PODCI (Child Parent-reported)",
    ]);
  });


  it('tracks the assembled questionnaire as the current and last step', () => {
    expect(component.currentStep?.key).toBe('PODCI_CH');
    expect(component.currentIndex).toBe(0);
    expect(component.isLastQuestionnaire).toBe(true);
    expect(component.remainingCount).toBe(0);
  });

  it('opens and cancels the Save & Close confirmation', () => {
    component.requestSaveAndClose();
    expect(component.showClosePopup).toBe(true);
    component.continueQuestionnaire();
    expect(component.showClosePopup).toBe(false);
  });

  it('uses the child PODCI variant and does not enable history-only sections', () => {
    expect(component.podciVariant('PODCI_CH')).toBe('CH');
    expect(component.showFirstVisit).toBe(false);
    expect(component.showGait).toBe(false);
  });
});








import { vi } from 'vitest';
import { QuestionnaireWelcome } from './questionnaire-welcome';

describe('QuestionnaireWelcome', () => {
  it('emits the start event when requested', () => {
    const component = new QuestionnaireWelcome();
    const start = vi.fn();
    component.startQuestionnaire.subscribe(start);
    component.start();
    expect(start).toHaveBeenCalledOnce();
  });
});






import { vi } from 'vitest';
import { provideHttpClient } from '@angular/common/http';
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';
import { TestBed } from '@angular/core/testing';
import { SportsQuestionnaire } from './sports-questionnaire';

describe('SportsQuestionnaire', () => {
  let component: SportsQuestionnaire;
  let http: HttpTestingController;
  beforeEach(async () => {
    await TestBed.configureTestingModule({ imports: [SportsQuestionnaire], providers: [provideHttpClient(), provideHttpClientTesting()] })
      .overrideComponent(SportsQuestionnaire, { set: { template: '' } }).compileComponents();
    component = TestBed.createComponent(SportsQuestionnaire).componentInstance;
    http = TestBed.inject(HttpTestingController);
  });
  afterEach(() => http.verify());
  it('requires reporting person, sport details, pain and goals', () => {
    expect(component.getInvalidFields()).toContain('reportingPerson');
    expect(component.getInvalidFields()).toContain('painLocations');
    expect(component.validate()).not.toBeNull();
  });
  it('treats None as mutually exclusive with individual pain locations', () => {
    component.togglePainLocation('R Hip');
    expect(component.answers.painLocations).toEqual(['R Hip']);
    component.togglePainLocation('None');
    expect(component.answers.painLocations).toEqual(['None']);
    component.togglePainLocation('L Hip');
    expect(component.answers.painLocations).toEqual(['None']);
    component.togglePainLocation('None');
    expect(component.answers.painLocations).toEqual([]);
  });
  it('clears the mile time when changed to Unknown', () => {
    component.setMileTimeType('Time');
    expect(component.isMileTimeEnabled()).toBe(true);
    component.answers.mileTime = '08:30';
    component.setMileTimeType('Unknown');
    expect(component.answers.mileTime).toBe('');
    expect(component.answers.mileTimeKnown).toBe('unknown');
  });
  it('refuses to save without a server-assigned questionnaire session', () => {
    component.patientId = 42;
    component.visitId = 10;
    const failed = vi.fn();
    component.saveFailed.subscribe(failed);
    component.save();
    expect(failed).toHaveBeenCalledOnce();
    http.expectNone('/api/patients/42/sports-questionnaire');
  });
});








import { vi } from 'vitest';
import { provideHttpClient } from '@angular/common/http';
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';
import { TestBed } from '@angular/core/testing';
import { UeHistoryQuestionnaire } from './ue-history-questionnaire';

describe('UeHistoryQuestionnaire', () => {
  let component: UeHistoryQuestionnaire;
  let http: HttpTestingController;
  beforeEach(async () => {
    await TestBed.configureTestingModule({ imports: [UeHistoryQuestionnaire], providers: [provideHttpClient(), provideHttpClientTesting()] })
      .overrideComponent(UeHistoryQuestionnaire, { set: { template: '' } }).compileComponents();
    component = TestBed.createComponent(UeHistoryQuestionnaire).componentInstance;
    http = TestBed.inject(HttpTestingController);
  });
  afterEach(() => http.verify());
  it('requires hand-use and three self-care answers', () => {
    expect(component.getProgress()).toEqual({ answered: 0, total: 4 });
    expect(component.getInvalidFields()).toEqual(['handlesObjects', 'dresses', 'bathes', 'toilets']);
  });
  it('tracks completed responses', () => {
    component.choose('handlesObjects', 'Yes');
    component.choose('dresses', 'No');
    component.choose('bathes', 'Yes');
    component.choose('toilets', 'Yes');
    expect(component.getProgress()).toEqual({ answered: 4, total: 4 });
    expect(component.validate()).toBeNull();
  });
  it('toggles upper-extremity concern choices without duplicating them', () => {
    component.toggleConcern('Weak grasp');
    component.toggleConcern('Weak grasp');
    expect(component.concerns.size).toBe(0);
  });
  it('does not save until a session was obtained from the server', () => {
    component.patientId = 42;
    component.visitId = 10;
    const failed = vi.fn();
    component.saveFailed.subscribe(failed);
    component.save();
    expect(failed).toHaveBeenCalledOnce();
    http.expectNone('/api/patients/42/ue-history-questionnaire');
  });
});









import { fillName, translate } from './legacy-i18n';

describe('Legacy questionnaire translation', () => {
  it('keeps English text and strips context suffixes', () => {
    expect(translate('Name#entry', 'en')).toBe('Name');
  });
  it('uses known Spanish translations', () => {
    expect(translate('Thank You', 'sp')).toBe('Gracias');
  });
  it('falls back to English for missing Spanish keys', () => {
    expect(translate('Untranslated sentence', 'sp')).toBe('Untranslated sentence');
  });
  it('replaces every patient-name placeholder using the supplied fallback', () => {
    expect(fillName('[name] and [name]', 'Alex', 'your child')).toBe('Alex and Alex');
    expect(fillName('Hello [name]', '', 'your child')).toBe('Hello your child');
  });
});







import { provideHttpClient } from "@angular/common/http";
import { HttpTestingController, provideHttpClientTesting } from "@angular/common/http/testing";
import { ComponentFixture, TestBed } from "@angular/core/testing";
import { provideRouter } from "@angular/router";

import { Questionnaire, QuestionnaireScheduledVisit } from "./questionnaire";

describe("Questionnaire", () => {
  let component: Questionnaire;
  let fixture: ComponentFixture<Questionnaire>;
  let httpTesting: HttpTestingController;

  const visit: QuestionnaireScheduledVisit = {
    visitId: 10,
    patientId: 1,
    firstName: "Avery",
    lastName: "Patient",
    mrn: "1234",
    visitType: null,
    visitSubType: null,
  };

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [Questionnaire],
      providers: [
        provideRouter([]),
        provideHttpClient(),
        provideHttpClientTesting(),
      ],
    }).compileComponents();

    httpTesting = TestBed.inject(HttpTestingController);
    fixture = TestBed.createComponent(Questionnaire);
    component = fixture.componentInstance;
    fixture.detectChanges();
    httpTesting
      .expectOne((request) => request.url === "/api/patients/scheduled")
      .flush([]);
    httpTesting.expectOne("/api/auth/me").flush(
      {},
      { status: 401, statusText: "Unauthorized" },
    );
  });

  afterEach(() => httpTesting.verify());

  it("should create", () => {
    expect(component).toBeTruthy();
  });

  it("should default to English and today's date", () => {
    expect(component.selectedLanguage).toBe("English");
    expect(component.selectedDate).toMatch(/^\d{4}-\d{2}-\d{2}$/);
  });

  it("should change the calendar year without changing the month or selected date", () => {
    const selectedDate = component.selectedDate;
    const month = component.calendarMonth.getMonth();

    component.changeYear(2016);

    expect(component.calendarMonth.getFullYear()).toBe(2016);
    expect(component.calendarMonth.getMonth()).toBe(month);
    expect(component.selectedDate).toBe(selectedDate);
    expect(component.calendarYears).toContain(2016);
    expect(component.calendarDays.some((day) => day.date.startsWith("2016-"))).toBe(
      true,
    );
  });

  it("should expose the available questionnaire categories", () => {
    const questionnaireIds = component.categories.flatMap((category) =>
      category.options.map((option) => option.id),
    );

    expect(questionnaireIds).toEqual([
      "history",
      "podci-parent",
      "podci-self",
      "sports",
      "hip",
    ]);
  });

  it("should select a visit and clear questionnaire selections", () => {
    component.selectVisit(visit);
    component.toggleQuestionnaire("history");
    component.selectVisit({ ...visit, visitId: 11 });

    expect(component.selectedPatientId).toBe(1);
    expect(component.selectedVisitId).toBe(11);
    expect(component.selectedQuestionnaireIds).toEqual([]);
  });

  it("should toggle questionnaire selections only after selecting a visit", () => {
    component.toggleQuestionnaire("history");
    expect(component.selectedQuestionnaireIds).toEqual([]);

    component.selectVisit(visit);
    component.toggleQuestionnaire("history");
    expect(component.selectedQuestionnaireIds).toEqual(["history"]);

    component.toggleQuestionnaire("history");
    expect(component.selectedQuestionnaireIds).toEqual([]);
  });

  it("should require a questionnaire before starting", () => {
    component.selectVisit(visit);
    component.startQuestionnaire();

    expect(component.errorMessage).toBe(
      "Please select at least one questionnaire.",
    );
  });
});










import { ComponentFixture, TestBed } from '@angular/core/testing';
import { provideHttpClient } from '@angular/common/http';
import {
  HttpTestingController,
  provideHttpClientTesting
} from '@angular/common/http/testing';

import { ScheduleVisit } from './schedule-visit';

describe('ScheduleVisit', () => {
  let component: ScheduleVisit;
  let fixture: ComponentFixture<ScheduleVisit>;
  let httpTesting: HttpTestingController;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [ScheduleVisit],
      providers: [
        provideHttpClient(),
        provideHttpClientTesting()
      ]
    }).compileComponents();

    fixture = TestBed.createComponent(ScheduleVisit);
    component = fixture.componentInstance;
    httpTesting = TestBed.inject(HttpTestingController);

    fixture.detectChanges();

    const endpoints = [
      '/api/schedule-visit/visit-types',
      '/api/schedule-visit/visit-sub-types',
      '/api/schedule-visit/tests',
      '/api/schedule-visit/referring-doctors'
    ];

    for (const endpoint of endpoints) {
      const request = httpTesting.expectOne(endpoint);
      expect(request.request.method).toBe('GET');
      request.flush([]);
    }
  });

  afterEach(() => {
    httpTesting.verify();
  });

  it('should create', () => {
    expect(component).toBeTruthy();
  });


  it('offers half-hour appointment choices', () => {
    expect(component.timeOptions.length).toBeGreaterThan(0);
    expect(component.timeOptions).toContain('8:00 AM');
  });

  it('rejects missing date or visit type', () => {
    component.patientId = 42;
    component.form.visitDate = '';
    component.saveForm();
    expect(component.errorMessage).toBe('Visit date is required.');
    component.form.visitDate = '2026-10-13';
    component.saveForm();
    expect(component.errorMessage).toBe('Visit type is required.');
    httpTesting.expectNone(request => request.method === 'POST');
  });

  it('rejects an end time before the start time', () => {
    component.patientId = 42;
    component.form.visitDate = '2026-10-13';
    component.form.visitTypeId = 1;
    component.form.startTime = '10:00 AM';
    component.form.endTime = '9:00 AM';
    component.saveForm();
    expect(component.errorMessage).toBe('End time must be after start time.');
    httpTesting.expectNone(request => request.method === 'POST');
  });

  it('loads default tests associated with the selected subtype', () => {
    component.form.selectedTests = ['stale'];
    component.onVisitSubTypeChange(3);
    const req = httpTesting.expectOne('/api/schedule-visit/visit-sub-types/3/default-tests');
    expect(req.request.method).toBe('GET');
    req.flush(['PT', 'GAIT']);
    expect(component.form.selectedTests).toEqual(['PT', 'GAIT']);
  });

  it('persists selected tests and emits the saved form after success', () => {
    component.patientId = 42;
    component.form.visitDate = '2026-10-13';
    component.form.visitTypeId = 1;
    component.form.selectedTests = ['GAIT'];
    const emitted: string[][] = [];
    component.save.subscribe(form => emitted.push(form.selectedTests));
    component.saveForm();
    const req = httpTesting.expectOne('/api/schedule-visit/patients/42');
    expect(req.request.method).toBe('POST');
    expect(req.request.body.selectedTests).toEqual(['GAIT']);
    req.flush({});
    expect(emitted).toEqual([['GAIT']]);
  });
});








import { provideHttpClient } from '@angular/common/http';
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';
import { TestBed } from '@angular/core/testing';
import { Router, provideRouter } from '@angular/router';
import { of } from 'rxjs';
import { vi } from 'vitest';
import { PatientService } from '../patients/patient.service';
import { Schedule, ScheduledVisit } from './schedule';

const VISIT: ScheduledVisit = {
  id: 1, patientId: 42, firstName: 'Alex', lastName: 'Smith', mrn: '500',
  visitDate: '2026-10-13', startTime: '9:00 AM', endTime: '10:00 AM', visitType: 'Gait', visitSubType: 'Initial',
};

describe('Schedule', () => {
  let component: Schedule;
  let http: HttpTestingController;
  let router: Router;
  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [Schedule],
      providers: [provideRouter([]), provideHttpClient(), provideHttpClientTesting(),
        { provide: PatientService, useValue: { searchPatients: vi.fn(() => of([])) } }],
    }).overrideComponent(Schedule, { set: { template: '' } }).compileComponents();
    component = TestBed.createComponent(Schedule).componentInstance;
    http = TestBed.inject(HttpTestingController);
    router = TestBed.inject(Router);
  });
  afterEach(() => { component.ngOnDestroy(); http.verify(); });

  it('builds a 42-day calendar with exactly one selected date', () => {
    component.buildCalendar();
    expect(component.calendarDays).toHaveLength(42);
    expect(component.calendarDays.filter(day => day.selected)).toHaveLength(1);
  });
  it('renders appointment positions and widths based on time and duration', () => {
    expect(component.appointmentTop(VISIT)).toBe(75);
    expect(component.appointmentHeight(VISIT)).toBe(75);
    expect(component.getPatientDisplayName(VISIT)).toBe('Alex Smith');
    expect(component.getVisitDisplayType(VISIT)).toBe('Gait - Initial');
  });
  it('loads scheduled visits using the selected date', () => {
    component.selectedDate = new Date(2026, 9, 13);
    component.loadVisits();
    const req = http.expectOne(request => request.url === '/api/schedule-visit/scheduled-visits');
    expect(req.request.params.get('date')).toBe('2026-10-13');
    req.flush([]);
    expect(component.scheduledVisits).toEqual([]);
    expect(component.loadingVisits).toBe(false);
  });
  it('displays a useful load error', () => {
    component.loadVisits();
    http.expectOne(request => request.url === '/api/schedule-visit/scheduled-visits').flush(null, { status: 500, statusText: 'Failure' });
    expect(component.scheduleError).toBe('Unable to load scheduled visits.');
  });
  it('opens the corresponding patient without navigating for an invalid ID', () => {
    const navigate = vi.spyOn(router, 'navigate').mockResolvedValue(true);
    component.openPatient({ ...VISIT, patientId: 0 });
    expect(navigate).not.toHaveBeenCalled();
    component.openPatient(VISIT);
    expect(navigate).toHaveBeenCalledWith(['/patients'], { queryParams: { patientId: 42 } });
  });
});







import { TestBed } from '@angular/core/testing';
import { HttpClient } from '@angular/common/http';
import { Router } from '@angular/router';
import { appConfig } from './app.config';

describe('Application provider configuration', () => {
  it('registers HttpClient and the router for application-wide use', () => {
    TestBed.configureTestingModule({ providers: [...appConfig.providers] });
    expect(TestBed.inject(HttpClient)).toBeTruthy();
    expect(TestBed.inject(Router)).toBeTruthy();
  });
});








import { routes } from './app.routes';
import { adminGuard, authGuard } from './core/auth.guard';

describe('Application routes', () => {
  it('protects every clinical feature route with authGuard', () => {
    for (const path of ['patients', 'questionnaire', 'questionnaire/session', 'forms', 'schedule', 'profile']) {
      const route = routes.find(item => item.path === path);
      expect(route?.canActivate, `route ${path}`).toContain(authGuard);
    }
  });
  it('protects admin with adminGuard rather than merely hiding its menu', () => {
    expect(routes.find(route => route.path === 'admin')?.canActivate).toContain(adminGuard);
  });
  it('has safe root and unknown-path fallbacks to login', () => {
    expect(routes.find(route => route.path === '')?.redirectTo).toBe('login');
    expect(routes.find(route => route.path === '**')?.redirectTo).toBe('login');
  });
});








import { TestBed } from "@angular/core/testing";
import { App } from "./app";

describe("App", () => {
  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [App],
    }).compileComponents();
  });

  it("should create the app", () => {
    const fixture = TestBed.createComponent(App);
    const app = fixture.componentInstance;
    expect(app).toBeTruthy();
  });
});

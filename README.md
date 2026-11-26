package org.nemours.gaitlab.controller;

import org.nemours.gaitlab.requests.SavePtEvaluationFormRequest;
import org.nemours.gaitlab.service.OrthosesService;
import org.nemours.gaitlab.service.PtEvaluationActions;
import org.nemours.gaitlab.service.PtEvaluationForm;
import org.nemours.gaitlab.service.PtEvaluationTab;
import org.springframework.http.HttpStatus;
import org.springframework.security.core.Authentication;
import org.springframework.web.bind.annotation.DeleteMapping;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.server.ResponseStatusException;

import jakarta.validation.Valid;

/** Ports insert_orth.jsp/modify_orth.jsp/delete_orth.jsp. All three write operations are
 * admin-only in legacy (the Add/Edit/Delete controls are only rendered for admins at all). */
@RestController
@RequestMapping("/api/patients/{patientId}/visits/{visitId}/pt-evaluation/orthotics")
public class OrthosesController {
    private final OrthosesService service;

    public OrthosesController(OrthosesService service) {
        this.service = service;
    }

    @GetMapping("/new/form")
    public PtEvaluationForm newForm(@PathVariable int patientId, @PathVariable int visitId) {
        return service.buildNewForm(visitId);
    }

    @PostMapping("/new/form")
    public PtEvaluationTab create(@PathVariable int patientId, @PathVariable int visitId,
            @Valid @RequestBody SavePtEvaluationFormRequest request, Authentication authentication) {
        boolean isAdmin = PtEvaluationActions.isAdmin(authentication);
        requireAdmin(isAdmin);
        return service.createOrthosis(patientId, visitId, isAdmin, request.revision(), request.values());
    }

    @GetMapping("/{id}/form")
    public PtEvaluationForm editForm(@PathVariable int patientId, @PathVariable int visitId, @PathVariable int id) {
        return service.buildEditForm(visitId, id);
    }

    @PutMapping("/{id}/form")
    public PtEvaluationTab update(@PathVariable int patientId, @PathVariable int visitId, @PathVariable int id,
            @Valid @RequestBody SavePtEvaluationFormRequest request, Authentication authentication) {
        boolean isAdmin = PtEvaluationActions.isAdmin(authentication);
        requireAdmin(isAdmin);
        return service.updateOrthosis(patientId, visitId, id, isAdmin, request.revision(), request.values());
    }

    @DeleteMapping("/{id}")
    public PtEvaluationTab delete(@PathVariable int patientId, @PathVariable int visitId, @PathVariable int id,
            Authentication authentication) {
        boolean isAdmin = PtEvaluationActions.isAdmin(authentication);
        requireAdmin(isAdmin);
        return service.deleteOrthosis(patientId, visitId, id, isAdmin);
    }

    private static void requireAdmin(boolean isAdmin) {
        if (!isAdmin) {
            throw new ResponseStatusException(HttpStatus.FORBIDDEN);
        }
    }
}














package org.nemours.gaitlab.controller;

import org.nemours.gaitlab.requests.SavePtEvaluationFormRequest;
import org.nemours.gaitlab.service.ComparisonReport;
import org.nemours.gaitlab.service.ComparisonReportService;
import org.nemours.gaitlab.service.GeneralMdService;
import org.nemours.gaitlab.service.PtEvaluationActions;
import org.nemours.gaitlab.service.PtEvaluationForm;
import org.nemours.gaitlab.service.PtEvaluationService;
import org.nemours.gaitlab.service.PtEvaluationService.GeneralEvaluation;
import org.nemours.gaitlab.service.PtEvaluationTab;
import org.springframework.http.HttpStatus;
import org.springframework.security.core.Authentication;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.server.ResponseStatusException;

import jakarta.validation.Valid;

@RestController
@RequestMapping("/api/patients/{patientId}/visits/{visitId}/pt-evaluation")
public class PtEvaluationController {
    private final PtEvaluationService service;
    private final ComparisonReportService comparisonReportService;
    private final GeneralMdService generalMdService;

    public PtEvaluationController(PtEvaluationService service, ComparisonReportService comparisonReportService,
            GeneralMdService generalMdService) {
        this.service = service;
        this.comparisonReportService = comparisonReportService;
        this.generalMdService = generalMdService;
    }

    @GetMapping("/comparison-report")
    public ComparisonReport comparisonReport(@PathVariable int patientId, @PathVariable int visitId,
            @RequestParam(required = false) Integer compareVisitId1,
            @RequestParam(required = false) Integer compareVisitId2) {
        return comparisonReportService.build(patientId, visitId, compareVisitId1, compareVisitId2);
    }

    @GetMapping("/general")
    public GeneralEvaluation general(@PathVariable int patientId, @PathVariable int visitId,
            Authentication authentication) {
        boolean isAdmin = PtEvaluationActions.isAdmin(authentication);
        // Gait-only for now: movement-disorder routing (and its own hidden "report" rule) lands
        // in Phase 4, so the Comparison Report button is always shown here.
        GeneralEvaluation evaluation = service.getGeneral(patientId, visitId);
        return evaluation.withActions(new PtEvaluationActions(isAdmin, isAdmin, false, true));
    }

    @GetMapping("/general/form")
    public PtEvaluationForm generalForm(@PathVariable int patientId, @PathVariable int visitId) {
        return service.buildForm(patientId, visitId);
    }

    @PutMapping("/general/form")
    public GeneralEvaluation saveGeneralForm(@PathVariable int patientId, @PathVariable int visitId,
            @Valid @RequestBody SavePtEvaluationFormRequest request, Authentication authentication) {
        boolean isAdmin = PtEvaluationActions.isAdmin(authentication);
        if (!isAdmin) {
            throw new ResponseStatusException(HttpStatus.FORBIDDEN);
        }
        GeneralEvaluation evaluation = service.save(patientId, visitId, request.revision(), request.values());
        return evaluation.withActions(new PtEvaluationActions(isAdmin, isAdmin, false, true));
    }

    @PostMapping("/general/record")
    public GeneralEvaluation createGeneralRecord(@PathVariable int patientId, @PathVariable int visitId,
            Authentication authentication) {
        boolean isAdmin = PtEvaluationActions.isAdmin(authentication);
        if (!isAdmin) {
            throw new ResponseStatusException(HttpStatus.FORBIDDEN);
        }
        GeneralEvaluation evaluation = service.createRecord(patientId, visitId);
        return evaluation.withActions(new PtEvaluationActions(isAdmin, isAdmin, false, true));
    }

    // --- Phase 4: lab/data/pt.jsp's isMovementDisorder branch ("General" tab, key "general-md") -

    @GetMapping("/general-md")
    public PtEvaluationTab generalMd(@PathVariable int patientId, @PathVariable int visitId,
            Authentication authentication) {
        boolean isAdmin = PtEvaluationActions.isAdmin(authentication);
        // pt.jsp hides "View Printable Comparison Report" for movement-disorder visits outright.
        PtEvaluationTab result = generalMdService.getGeneral(patientId, visitId, "general-md");
        return result.withActions(new PtEvaluationActions(isAdmin, isAdmin, false, false));
    }

    @GetMapping("/general-md/form")
    public PtEvaluationForm generalMdForm(@PathVariable int patientId, @PathVariable int visitId) {
        return generalMdService.buildForm(patientId, visitId);
    }

    @PutMapping("/general-md/form")
    public PtEvaluationTab saveGeneralMdForm(@PathVariable int patientId, @PathVariable int visitId,
            @Valid @RequestBody SavePtEvaluationFormRequest request, Authentication authentication) {
        boolean isAdmin = PtEvaluationActions.isAdmin(authentication);
        if (!isAdmin) {
            throw new ResponseStatusException(HttpStatus.FORBIDDEN);
        }
        PtEvaluationTab result = generalMdService.save(patientId, visitId, request.revision(), request.values());
        return result.withActions(new PtEvaluationActions(isAdmin, isAdmin, false, false));
    }

    @PostMapping("/general-md/record")
    public PtEvaluationTab createGeneralMdRecord(@PathVariable int patientId, @PathVariable int visitId,
            Authentication authentication) {
        boolean isAdmin = PtEvaluationActions.isAdmin(authentication);
        if (!isAdmin) {
            throw new ResponseStatusException(HttpStatus.FORBIDDEN);
        }
        PtEvaluationTab result = generalMdService.createRecord(patientId, visitId);
        return result.withActions(new PtEvaluationActions(isAdmin, isAdmin, false, false));
    }
}
















package org.nemours.gaitlab.controller;

import org.nemours.gaitlab.service.PtEvaluationValidationException;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

/** Shared 400 response for PtEvaluationValidationException across the PT Evaluation controllers
 * (PtEvaluationController, PtEvaluationTabsController, OrthosesController), which all threw the
 * same validation exception for their respective save endpoints. */
@RestControllerAdvice
public class PtEvaluationExceptionHandler {
    @ExceptionHandler(PtEvaluationValidationException.class)
    public ResponseEntity<PtEvaluationValidationException.Response> onValidationError(
            PtEvaluationValidationException exception) {
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
                .body(new PtEvaluationValidationException.Response(exception.getMessage(), exception.fieldErrors()));
    }
}














package org.nemours.gaitlab.controller;

import org.nemours.gaitlab.requests.SavePtEvaluationFormRequest;
import org.nemours.gaitlab.service.PtEvaluationActions;
import org.nemours.gaitlab.service.PtEvaluationForm;
import org.nemours.gaitlab.service.PtEvaluationTab;
import org.nemours.gaitlab.service.PtEvaluationTabList;
import org.nemours.gaitlab.service.PtEvaluationTabsService;
import org.springframework.security.core.Authentication;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import jakarta.validation.Valid;

@RestController
@RequestMapping("/api/patients/{patientId}/visits/{visitId}/pt-evaluation")
public class PtEvaluationTabsController {
    private final PtEvaluationTabsService service;

    public PtEvaluationTabsController(PtEvaluationTabsService service) {
        this.service = service;
    }

    @GetMapping("/tabs")
    public PtEvaluationTabList tabs(@PathVariable int patientId, @PathVariable int visitId) {
        return service.getTabs(patientId, visitId);
    }

    @GetMapping("/{tab}")
    public PtEvaluationTab tab(@PathVariable int patientId, @PathVariable int visitId, @PathVariable String tab,
            Authentication authentication) {
        return service.get(patientId, visitId, tab, PtEvaluationActions.isAdmin(authentication));
    }

    @GetMapping("/{tab}/form")
    public PtEvaluationForm form(@PathVariable int patientId, @PathVariable int visitId, @PathVariable String tab) {
        return service.getForm(patientId, visitId, tab);
    }

    @PutMapping("/{tab}/form")
    public PtEvaluationTab saveForm(@PathVariable int patientId, @PathVariable int visitId,
            @PathVariable String tab, @Valid @RequestBody SavePtEvaluationFormRequest request,
            Authentication authentication) {
        return service.saveForm(patientId, visitId, tab, PtEvaluationActions.isAdmin(authentication), request);
    }

    @PostMapping("/{tab}/record")
    public PtEvaluationTab createRecord(@PathVariable int patientId, @PathVariable int visitId,
            @PathVariable String tab, Authentication authentication) {
        return service.createRecord(patientId, visitId, tab, PtEvaluationActions.isAdmin(authentication));
    }
}
















package org.nemours.gaitlab.repository;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Optional;

import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Repository;

/** Single data-access class for the whole PT Evaluation feature - one repository per tab/table
 * was judged to be more classes than the feature warranted, so Dystonia/HAT/General/General-MD/
 * Motor Control-Tone/Orthoses/Pain/PROM-Strength are all merged in here. Method names are
 * prefixed with their domain only where the original per-tab class names would otherwise collide
 * (Dystonia/HAT both had findItems/findSelectionsForItem/findSelectionById/
 * existsSelectionForItem; Orthoses/Pain/PROM-Strength all had a findByVisitId with a different
 * table behind it); every other method keeps its original, already-unique name. */
@Repository
public class PtEvaluationRepository {
    public static final List<String> ULLR_OPTIONS = List.of("UER", "LER", "UEL", "LEL");

    private final JdbcTemplate jdbcTemplate;

    public PtEvaluationRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    // --- Dystonia Scale: pt-eval-dystonia-scale.jsp/modify_pt_dystonia_scale.jsp ---------------
    // Six items (dystonia_item, value 1-6: Eyes/Mouth/Neck/Trunk/UE/LE); Eyes/Mouth/Neck/Trunk are
    // unilateral (dystonia_scale.eyes/mouth/neck/trunk), UE/LE are bilateral (uer/uel, ler/lel) -
    // matches DystoniaItem#isBilateral() and DystoniaScale#getSelection(item, side) in the legacy
    // Java model.

    // Frontend: PT Evaluation - Dystonia Scale tab (item list)
    public List<Map<String, Object>> findDystoniaItems() {
        return jdbcTemplate.queryForList(
                "SELECT id, item, longdesc, value FROM public.dystonia_item WHERE active = TRUE ORDER BY value ASC");
    }

    // Frontend: PT Evaluation - Dystonia Scale tab (selection dropdown)
    public List<Map<String, Object>> findDystoniaSelectionsForItem(int itemId) {
        return jdbcTemplate.queryForList("""
                SELECT id, item AS selection, value, sortorder FROM public.dystonia_selection
                WHERE item_id = ? AND active = TRUE ORDER BY sortorder ASC
                """, itemId);
    }

    // Frontend: PT Evaluation - Dystonia Scale tab
    public Optional<Map<String, Object>> findDystoniaScaleByVisitId(int visitId) {
        List<Map<String, Object>> rows = jdbcTemplate.queryForList(
                "SELECT * FROM public.dystonia_scale WHERE visit_id = ?", visitId);
        return rows.isEmpty() ? Optional.empty() : Optional.of(rows.get(0));
    }

    // Frontend: PT Evaluation - Dystonia Scale tab
    public Optional<Map<String, Object>> findDystoniaSelectionById(int selectionId) {
        List<Map<String, Object>> rows = jdbcTemplate.queryForList(
                "SELECT id, item AS selection, value FROM public.dystonia_selection WHERE id = ?", selectionId);
        return rows.isEmpty() ? Optional.empty() : Optional.of(rows.get(0));
    }

    // Frontend: PT Evaluation - Dystonia Scale tab (save validation)
    public boolean existsDystoniaSelectionForItem(int itemId, int selectionId) {
        return !jdbcTemplate.queryForList(
                "SELECT 1 FROM public.dystonia_selection WHERE id = ? AND item_id = ?", selectionId, itemId)
                .isEmpty();
    }

    // Frontend: PT Evaluation - Dystonia Scale tab (save form)
    public void updateDystoniaScale(int id, Integer eyes, Integer mouth, Integer neck, Integer trunk, Integer uel,
            Integer uer, Integer lel, Integer ler) {
        jdbcTemplate.update("""
                UPDATE public.dystonia_scale
                SET eyes = ?, mouth = ?, neck = ?, trunk = ?, uel = ?, uer = ?, lel = ?, ler = ?
                WHERE id = ?
                """, eyes, mouth, neck, trunk, uel, uer, lel, ler, id);
    }

    // Frontend: PT Evaluation - Dystonia Scale tab (Create New Record)
    public int insertBlankDystoniaScale(int visitId) {
        return jdbcTemplate.queryForObject(
                "INSERT INTO public.dystonia_scale (visit_id) VALUES (?) RETURNING id", Integer.class, visitId);
    }

    // --- General (Gait): modify_pt.jsp's writes (the General & GMFM edit form) ----------------
    // height/weight/foot measurements live on visits; everything else lives on data_pt, so a save
    // touches both tables - ported as two separate statements, same as legacy's own two
    // PreparedStatements.

    // Frontend: PT Evaluation - General & GMFM tab, and General tab (movement-disorder visits)
    public List<Map<String, Object>> findActiveGmfcsListEntries() {
        return jdbcTemplate.queryForList(
                "SELECT id, label FROM public.gmfcs_list WHERE active = TRUE ORDER BY sortorder ASC");
    }

    // Frontend: PT Evaluation - General & GMFM tab, and General tab (movement-disorder visits)
    public List<Map<String, Object>> findActiveFmsListEntries() {
        return jdbcTemplate.queryForList(
                "SELECT id, label FROM public.fms_list WHERE active = TRUE ORDER BY sortorder ASC");
    }

    // Frontend: PT Evaluation - General & GMFM tab (save validation)
    public boolean existsGmfcsListEntry(int id) {
        return !jdbcTemplate.queryForList("SELECT 1 FROM public.gmfcs_list WHERE id = ?", id).isEmpty();
    }

    // Frontend: PT Evaluation - General & GMFM tab (save validation)
    public boolean existsFmsListEntry(int id) {
        return !jdbcTemplate.queryForList("SELECT 1 FROM public.fms_list WHERE id = ?", id).isEmpty();
    }

    // Frontend: PT Evaluation - General & GMFM tab, and General tab (movement-disorder, save form)
    public void updateVisitMeasurements(int visitId, Float height, Float weight, Float rightFootLength,
            Float leftFootLength, Float rightFootWidth, Float leftFootWidth) {
        jdbcTemplate.update("""
                UPDATE public.visits
                SET height = ?, weight = ?, r_foot_length = ?, l_foot_length = ?, r_foot_width = ?, l_foot_width = ?
                WHERE id = ?
                """, height, weight, rightFootLength, leftFootLength, rightFootWidth, leftFootWidth, visitId);
    }

    // Frontend: PT Evaluation - General & GMFM tab (save form)
    public void updateDataPt(int dataPtId, Integer ambScale, Integer gmfcs, Integer fms5, Integer fms50,
            Integer fms500, String comments, Integer pullStand, Integer indepStand3, Integer indepStand20,
            Integer oneLegHandR, Integer oneLegHandL, Integer oneLegR, Integer oneLegL, Integer shortSitStand,
            Integer highKneeStandR, Integer highKneeStandL, Integer lowerFloor, Integer squats,
            Integer standPickPen, Integer gmfmModA, Integer gmfmModB, Integer gmfmModC, Integer gmfmModE) {
        jdbcTemplate.update("""
                UPDATE public.data_pt
                SET amb_scale = ?, gmfcs = ?, fms_5 = ?, fms_50 = ?, fms_500 = ?, comments = ?,
                    pull_stand = ?, indep_stand_3 = ?, indep_stand_20 = ?, one_leg_hand_r = ?, one_leg_hand_l = ?,
                    one_leg_r = ?, one_leg_l = ?, short_sit_stand = ?, high_knee_stand_r = ?, high_knee_stand_l = ?,
                    lower_floor = ?, squats = ?, stand_pick_pen = ?, gmfm_mod_a = ?, gmfm_mod_b = ?,
                    gmfm_mod_c = ?, gmfm_mod_e = ?
                WHERE id = ?
                """, ambScale, gmfcs, fms5, fms50, fms500, comments, pullStand, indepStand3, indepStand20,
                oneLegHandR, oneLegHandL, oneLegR, oneLegL, shortSitStand, highKneeStandR, highKneeStandL,
                lowerFloor, squats, standPickPen, gmfmModA, gmfmModB, gmfmModC, gmfmModE, dataPtId);
    }

    /** Mirrors pt.jsp's own "Create New Record" insert - a bare data_pt row, visit_id only. */
    // Frontend: PT Evaluation - General & GMFM tab (Create New Record)
    public int insertBlankDataPt(int visitId) {
        return jdbcTemplate.queryForObject(
                "INSERT INTO public.data_pt (visit_id) VALUES (?) RETURNING id", Integer.class, visitId);
    }

    // --- General (Movement Disorder): modify_pt.jsp's isMovementDisorder branch ---------------
    // amb_scale/comments/gmfcs/fms_5/fms_50/fms_500/macs/cfcs on data_pt_movement_disorder.
    // height/weight/foot measurements still live on visits and are shared with the Gait form -
    // updateVisitMeasurements above is reused for those rather than duplicated here.

    // Frontend: PT Evaluation - General tab (movement-disorder visits)
    public List<Map<String, Object>> findActiveMacsListEntries() {
        return jdbcTemplate.queryForList(
                "SELECT id, label FROM public.macs_list WHERE active = TRUE ORDER BY sortorder ASC");
    }

    // Frontend: PT Evaluation - General tab (movement-disorder visits)
    public List<Map<String, Object>> findActiveCfcsListEntries() {
        return jdbcTemplate.queryForList(
                "SELECT id, label FROM public.cfcs_list WHERE active = TRUE ORDER BY sortorder ASC");
    }

    // Frontend: PT Evaluation - General tab (movement-disorder, save validation)
    public boolean existsMacsListEntry(int id) {
        return !jdbcTemplate.queryForList("SELECT 1 FROM public.macs_list WHERE id = ?", id).isEmpty();
    }

    // Frontend: PT Evaluation - General tab (movement-disorder, save validation)
    public boolean existsCfcsListEntry(int id) {
        return !jdbcTemplate.queryForList("SELECT 1 FROM public.cfcs_list WHERE id = ?", id).isEmpty();
    }

    // Frontend: PT Evaluation - General tab (movement-disorder, save form)
    public void updateDataPtMovementDisorder(int dataPtMdId, Integer ambScale, Integer gmfcs, Integer fms5,
            Integer fms50, Integer fms500, Integer macs, Integer cfcs, String comments) {
        jdbcTemplate.update("""
                UPDATE public.data_pt_movement_disorder
                SET amb_scale = ?, gmfcs = ?, fms_5 = ?, fms_50 = ?, fms_500 = ?, macs = ?, cfcs = ?, comments = ?
                WHERE id = ?
                """, ambScale, gmfcs, fms5, fms50, fms500, macs, cfcs, comments, dataPtMdId);
    }

    /** Mirrors insertBlankDataPt - a bare data_pt_movement_disorder row, visit_id only
     * (gmfcs/fms_5/fms_50/fms_500 fall back to their NOT NULL DEFAULT 1). */
    // Frontend: PT Evaluation - General tab (movement-disorder, Create New Record)
    public int insertBlankDataPtMovementDisorder(int visitId) {
        return jdbcTemplate.queryForObject(
                "INSERT INTO public.data_pt_movement_disorder (visit_id) VALUES (?) RETURNING id", Integer.class,
                visitId);
    }

    /** pt-eval-orth.jsp's own MD-only "Comments" textarea+Save - touches only this one column,
     * unlike the General tab's own save which rewrites every field at once. Named apart from the
     * Orthoses-domain methods below since it touches data_pt_movement_disorder, not pt_orthoses. */
    // Frontend: PT Evaluation - Orthotics tab (movement-disorder visits, Comments field)
    public void updateMovementDisorderOrthosesComment(int dataPtMdId, String orthoses) {
        jdbcTemplate.update("UPDATE public.data_pt_movement_disorder SET orthoses = ? WHERE id = ?", orthoses,
                dataPtMdId);
    }

    // --- HAT: pt-eval-hat.jsp/modify_pt_hat.jsp ------------------------------------------------
    // The 7-item Hypertonia Assessment Tool, scored once per limb (hat_scores.ullr - UEL/UER/LEL/
    // LER, UNIQUE(visit_id, ullr) - production's actual schema, gaitdata.export.psql:3298-3310/
    // 8029; aligned here via an explicit migration, see the Phase 4 audit scratchpad). The
    // ULLR_OPTIONS order (UER, LER, UEL, LEL) is the column order both pt-eval-hat.jsp and
    // modify_pt_hat.jsp iterate in - matched exactly below.

    // Frontend: PT Evaluation - HAT tab (item list)
    public List<Map<String, Object>> findHatItems() {
        return jdbcTemplate.queryForList(
                "SELECT id, item, value FROM public.hat_item WHERE active = TRUE ORDER BY value ASC");
    }

    // Frontend: PT Evaluation - HAT tab (selection dropdown)
    public List<Map<String, Object>> findHatSelectionsForItem(int itemId) {
        return jdbcTemplate.queryForList(
                "SELECT id, selection, value FROM public.hat_selection WHERE item_id = ? AND active = TRUE ORDER BY value ASC",
                itemId);
    }

    /** 0-4 rows, each with its own "ullr" - however many limbs have been scored for this visit. */
    // Frontend: PT Evaluation - HAT tab
    public List<Map<String, Object>> findHatScoresByVisitId(int visitId) {
        return jdbcTemplate.queryForList("SELECT * FROM public.hat_scores WHERE visit_id = ?", visitId);
    }

    /** Resolves a hatN selection's value+selection text, or null if unset/unknown. */
    // Frontend: PT Evaluation - HAT tab
    public Optional<Map<String, Object>> findHatSelectionById(int selectionId) {
        List<Map<String, Object>> rows = jdbcTemplate.queryForList(
                "SELECT id, selection, value FROM public.hat_selection WHERE id = ?", selectionId);
        return rows.isEmpty() ? Optional.empty() : Optional.of(rows.get(0));
    }

    // Frontend: PT Evaluation - HAT tab (save validation)
    public boolean existsHatSelectionForItem(int itemId, int selectionId) {
        return !jdbcTemplate.queryForList(
                "SELECT 1 FROM public.hat_selection WHERE id = ? AND item_id = ?", selectionId, itemId).isEmpty();
    }

    // Frontend: PT Evaluation - HAT tab (save form)
    public void updateHatScores(int id, Integer hat1, Integer hat2, Integer hat3, Integer hat4, Integer hat5,
            Integer hat6, Integer hat7) {
        jdbcTemplate.update("""
                UPDATE public.hat_scores
                SET hat1 = ?, hat2 = ?, hat3 = ?, hat4 = ?, hat5 = ?, hat6 = ?, hat7 = ?
                WHERE id = ?
                """, hat1, hat2, hat3, hat4, hat5, hat6, hat7, id);
    }

    /** pt-eval-hat.jsp's own "Create New Record" handler creates all 4 ULLR_OPTIONS rows at once
     * (visit.setHatScores(scores) with one HatScore per limb), not just one. */
    // Frontend: PT Evaluation - HAT tab (Create New Record)
    public void insertBlankHatScoresForAllLimbs(int visitId) {
        for (String ullr : ULLR_OPTIONS) {
            jdbcTemplate.update(
                    "INSERT INTO public.hat_scores (visit_id, ullr) VALUES (?, ?)", visitId, ullr);
        }
    }

    // --- Motor Control / Tone: pt-eval-mc.jsp/pt-eval-tone.jsp ---------------------------------
    // Both join data_tone to a lookup table (mc_tests / tone_tests) by test name, ordered by the
    // lookup table's own id.

    // Frontend: PT Evaluation - Motor Control tab, and the Comparison Report
    public List<Map<String, Object>> findMotorControlByVisitId(int visitId) {
        return jdbcTemplate.queryForList("""
                SELECT dt.id, dt.test, dt.score_r, dt.interp_r, dt.score_l, dt.interp_l
                FROM public.data_tone dt
                JOIN public.mc_tests mt ON dt.test = mt.test
                WHERE dt.visit_id = ? AND dt.mc_tone = 'mc'
                ORDER BY mt.id
                """, visitId);
    }

    // Frontend: PT Evaluation - Tone tab (Gait visits), and the Comparison Report
    public List<Map<String, Object>> findToneByVisitId(int visitId) {
        return jdbcTemplate.queryForList("""
                SELECT dt.id, dt.test, dt.score_r, dt.interp_r, dt.score_l, dt.interp_l
                FROM public.data_tone dt
                JOIN public.tone_tests tt ON dt.test = tt.test
                WHERE dt.visit_id = ? AND dt.mc_tone = 'tone'
                ORDER BY tt.id
                """, visitId);
    }

    /** pt-movement-disorder-eval-tone.jsp's own test list: only tone_tests.id <= 16 (its loop
     * explicitly checks `t.getTest().getId() <= 16`) - Proximal/Distal Muscle Tone (100/101) and
     * *Clonus Severity Notes (102) are Gait-Tone-only, not shown on the movement-disorder page. */
    // Frontend: PT Evaluation - Tone tab (movement-disorder visits)
    public List<Map<String, Object>> findToneByVisitIdForMovementDisorder(int visitId) {
        return jdbcTemplate.queryForList("""
                SELECT dt.id, dt.test, dt.score_r, dt.interp_r, dt.score_l, dt.interp_l
                FROM public.data_tone dt
                JOIN public.tone_tests tt ON dt.test = tt.test
                WHERE dt.visit_id = ? AND dt.mc_tone = 'tone' AND tt.id <= 16
                ORDER BY tt.id
                """, visitId);
    }

    // Frontend: PT Evaluation - Motor Control tab (Create New Record)
    public List<String> findMcTestNames() {
        return jdbcTemplate.queryForList("SELECT test FROM public.mc_tests ORDER BY id ASC", String.class);
    }

    // Frontend: PT Evaluation - Tone tab (Create New Record)
    public List<String> findToneTestNames() {
        return jdbcTemplate.queryForList("SELECT test FROM public.tone_tests ORDER BY id ASC", String.class);
    }

    // Frontend: PT Evaluation - Motor Control tab (save-form score dropdown)
    public List<String> findMcListOptions() {
        return jdbcTemplate.queryForList("SELECT score FROM public.mc_list", String.class);
    }

    // Frontend: PT Evaluation - Tone tab (save-form score dropdown)
    public List<String> findToneListOptions() {
        return jdbcTemplate.queryForList("SELECT scale FROM public.tone_list", String.class);
    }

    /** Shared by Motor Control, Tone and PROM/Strength - all three read the exact same
     * "interp" lookup table the same way, so this one method now backs all of them instead of
     * three identical copies. */
    // Frontend: PT Evaluation - Motor Control, Tone, and PROM/Strength tabs (interpretation dropdown)
    public List<String> findInterpOptions() {
        return jdbcTemplate.queryForList("SELECT scale FROM public.interp ORDER BY id ASC", String.class);
    }

    // Frontend: PT Evaluation - Motor Control and Tone tabs (Create New Record)
    public void insertBlankRow(int visitId, String test, String mcTone) {
        jdbcTemplate.update(
                "INSERT INTO public.data_tone (visit_id, test, mc_tone) VALUES (?, ?, ?)", visitId, test, mcTone);
    }

    // Frontend: PT Evaluation - Motor Control and Tone tabs (save form)
    public void updateScoreAndInterp(int id, String scoreR, String interpR, String scoreL, String interpL) {
        jdbcTemplate.update(
                "UPDATE public.data_tone SET score_r = ?, interp_r = ?, score_l = ?, interp_l = ? WHERE id = ?",
                scoreR, interpR, scoreL, interpL, id);
    }

    // Frontend: PT Evaluation - Motor Control and Tone tabs (save form, interpretation-only tests)
    public void updateInterpOnly(int id, String interpR, String interpL) {
        jdbcTemplate.update(
                "UPDATE public.data_tone SET interp_r = ?, interp_l = ? WHERE id = ?", interpR, interpL, id);
    }

    /** pt-movement-disorder-eval-tone.jsp's Upper Extremity rows - one row per visit (unlike
     * data_tone's one-row-per-test), six muscle groups with R/L score + R/L interp each. */
    // Frontend: PT Evaluation - Tone tab (movement-disorder visits, Upper Extremity rows)
    public Optional<Map<String, Object>> findUeToneByVisitId(int visitId) {
        List<Map<String, Object>> rows = jdbcTemplate.queryForList(
                "SELECT * FROM public.pt_ue_tone WHERE visit_id = ?", visitId);
        return rows.isEmpty() ? Optional.empty() : Optional.of(rows.get(0));
    }

    private static final List<String> UE_TONE_COLUMNS = List.of(
            "shoulder_extensors_r", "shoulder_extensors_r_interp", "shoulder_extensors_l", "shoulder_extensors_l_interp",
            "shoulder_flexors_r", "shoulder_flexors_r_interp", "shoulder_flexors_l", "shoulder_flexors_l_interp",
            "elbow_extensors_r", "elbow_extensors_r_interp", "elbow_extensors_l", "elbow_extensors_l_interp",
            "elbow_flexors_r", "elbow_flexors_r_interp", "elbow_flexors_l", "elbow_flexors_l_interp",
            "wrist_extensors_r", "wrist_extensors_r_interp", "wrist_extensors_l", "wrist_extensors_l_interp",
            "wrist_flexors_r", "wrist_flexors_r_interp", "wrist_flexors_l", "wrist_flexors_l_interp");

    /** Dynamic SET clause built only from the fixed UE_TONE_COLUMNS whitelist above - never from
     * request-supplied keys - same SQL-injection-safe pattern as the PROM/Strength update below. */
    // Frontend: PT Evaluation - Tone tab (movement-disorder visits, save Upper Extremity rows)
    public void updateUeTone(int id, Map<String, String> columns) {
        List<String> setClauses = new ArrayList<>();
        List<Object> params = new ArrayList<>();
        for (String column : UE_TONE_COLUMNS) {
            if (columns.containsKey(column)) {
                setClauses.add(column + " = ?");
                params.add(columns.get(column));
            }
        }
        if (setClauses.isEmpty()) {
            return;
        }
        params.add(id);
        jdbcTemplate.update(
                "UPDATE public.pt_ue_tone SET " + String.join(", ", setClauses) + " WHERE id = ?",
                params.toArray());
    }

    // Frontend: PT Evaluation - Tone tab (movement-disorder visits, Create New Record)
    public int insertBlankUeTone(int visitId) {
        return jdbcTemplate.queryForObject(
                "INSERT INTO public.pt_ue_tone (visit_id) VALUES (?) RETURNING id", Integer.class, visitId);
    }

    // --- Orthoses: pt-eval-orth.jsp's orthoses list and insert_orth.jsp/modify_orth.jsp/ -------
    // delete_orth.jsp (the movement-disorder "Comments" block lives above, in the General-MD
    // section - it touches data_pt_movement_disorder, not pt_orthoses). pt_orthoses has no FK
    // children (no trial/device rows reference it), so delete is a plain single-row delete.

    // Frontend: PT Evaluation - Orthotics tab (list)
    public List<Map<String, Object>> findOrthosesByVisitId(int visitId) {
        return jdbcTemplate.queryForList(
                "SELECT id, orthosis, side FROM public.pt_orthoses WHERE visit_id = ? ORDER BY orthosis ASC",
                visitId);
    }

    // Frontend: PT Evaluation - Orthotics tab (edit form)
    public Optional<Map<String, Object>> findByIdAndVisitId(int id, int visitId) {
        return jdbcTemplate.queryForList(
                "SELECT id, orthosis, side FROM public.pt_orthoses WHERE id = ? AND visit_id = ?", id, visitId)
                .stream().findFirst();
    }

    // Frontend: PT Evaluation - Orthotics tab (orthosis-type dropdown)
    public List<String> findOrthosisOptions() {
        return jdbcTemplate.queryForList("SELECT orthosis FROM public.orthoses ORDER BY orthosis", String.class);
    }

    // Frontend: PT Evaluation - Orthotics tab (Add an Orthosis)
    public int insert(int visitId, String orthosis, String side) {
        return jdbcTemplate.queryForObject(
                "INSERT INTO public.pt_orthoses (visit_id, orthosis, side) VALUES (?, ?, ?) RETURNING id",
                Integer.class, visitId, orthosis, side);
    }

    // Frontend: PT Evaluation - Orthotics tab (edit an orthosis)
    public void update(int id, int visitId, String orthosis, String side) {
        jdbcTemplate.update(
                "UPDATE public.pt_orthoses SET orthosis = ?, side = ? WHERE id = ? AND visit_id = ?",
                orthosis, side, id, visitId);
    }

    // Frontend: PT Evaluation - Orthotics tab (delete an orthosis)
    public void deleteByIdAndVisitId(int id, int visitId) {
        jdbcTemplate.update("DELETE FROM public.pt_orthoses WHERE id = ? AND visit_id = ?", id, visitId);
    }

    // --- Pain: pt-eval-pain.jsp's read of the JPA DataPain entity and its two lookup-backed -----
    // lists.

    // Frontend: PT Evaluation - Pain tab
    public Optional<Map<String, Object>> findPainByVisitId(int visitId) {
        return jdbcTemplate.queryForList("""
                SELECT dp.id, dp.current_level, dp.highest_level, dp.lowest_level,
                       dp.location, dp.frequency, dp.duration,
                       dp.pain_assessment_method_id, pam.label AS assessment_method
                FROM public.data_pain dp
                LEFT JOIN public.pain_assessment_methods pam ON pam.id = dp.pain_assessment_method_id
                WHERE dp.visit_id = ?
                """, visitId).stream().findFirst();
    }

    // Frontend: PT Evaluation - Pain tab (pain factors list)
    public List<String> findFactorLabels(int dataPainId) {
        return jdbcTemplate.queryForList("""
                SELECT pf.label
                FROM public.data_pain_to_pain_factors link
                JOIN public.pain_factors pf ON pf.id = link.pain_factor_id
                WHERE link.data_pain_id = ?
                ORDER BY pf.sortorder
                """, String.class, dataPainId);
    }

    // Frontend: PT Evaluation - Pain tab (activities impacted list)
    public List<String> findActivitiesImpactedLabels(int dataPainId) {
        return jdbcTemplate.queryForList("""
                SELECT pa.label
                FROM public.data_pain_to_pain_activities_impacted link
                JOIN public.pain_activities_impacted pa ON pa.id = link.pain_activities_impacted_id
                WHERE link.data_pain_id = ?
                ORDER BY pa.sortorder
                """, String.class, dataPainId);
    }

    // Frontend: PT Evaluation - Pain tab (save form)
    public List<Integer> findFactorIds(int dataPainId) {
        return jdbcTemplate.queryForList(
                "SELECT pain_factor_id FROM public.data_pain_to_pain_factors WHERE data_pain_id = ?",
                Integer.class, dataPainId);
    }

    // Frontend: PT Evaluation - Pain tab (save form)
    public List<Integer> findActivitiesImpactedIds(int dataPainId) {
        return jdbcTemplate.queryForList(
                "SELECT pain_activities_impacted_id FROM public.data_pain_to_pain_activities_impacted"
                        + " WHERE data_pain_id = ?",
                Integer.class, dataPainId);
    }

    // Frontend: PT Evaluation - Pain tab (assessment method dropdown)
    public List<Map<String, Object>> findAssessmentMethodOptions() {
        return jdbcTemplate.queryForList(
                "SELECT id, label FROM public.pain_assessment_methods ORDER BY sortorder");
    }

    // Frontend: PT Evaluation - Pain tab (pain factors checklist)
    public List<Map<String, Object>> findFactorOptions() {
        return jdbcTemplate.queryForList("SELECT id, label FROM public.pain_factors ORDER BY sortorder");
    }

    // Frontend: PT Evaluation - Pain tab (activities impacted checklist)
    public List<Map<String, Object>> findActivitiesImpactedOptions() {
        return jdbcTemplate.queryForList(
                "SELECT id, label FROM public.pain_activities_impacted ORDER BY sortorder");
    }

    // Frontend: PT Evaluation - Pain tab (save validation)
    public boolean existsAssessmentMethod(int id) {
        return !jdbcTemplate.queryForList(
                "SELECT 1 FROM public.pain_assessment_methods WHERE id = ?", id).isEmpty();
    }

    // Frontend: PT Evaluation - Pain tab (save validation)
    public boolean existsFactor(int id) {
        return !jdbcTemplate.queryForList("SELECT 1 FROM public.pain_factors WHERE id = ?", id).isEmpty();
    }

    // Frontend: PT Evaluation - Pain tab (save validation)
    public boolean existsActivityImpacted(int id) {
        return !jdbcTemplate.queryForList("SELECT 1 FROM public.pain_activities_impacted WHERE id = ?", id)
                .isEmpty();
    }

    /** Inserts a blank data_pain row (every column besides visit_id left null) and returns its
     * new id, mirroring pt-eval-pain.jsp's "Create New Record" - a bare row, nothing else. */
    // Frontend: PT Evaluation - Pain tab (Create New Record)
    public int insertBlankRecord(int visitId) {
        return jdbcTemplate.queryForObject(
                "INSERT INTO public.data_pain (visit_id) VALUES (?) RETURNING id", Integer.class, visitId);
    }

    // Frontend: PT Evaluation - Pain tab (save form)
    public void updateRecord(int dataPainId, Integer assessmentMethodId, Integer currentLevel, Integer lowestLevel,
            Integer highestLevel, String location, String frequency, String duration) {
        jdbcTemplate.update("""
                UPDATE public.data_pain
                SET pain_assessment_method_id = ?, current_level = ?, lowest_level = ?, highest_level = ?,
                    location = ?, frequency = ?, duration = ?
                WHERE id = ?
                """, assessmentMethodId, currentLevel, lowestLevel, highestLevel, location, frequency, duration,
                dataPainId);
    }

    // Frontend: PT Evaluation - Pain tab (save form)
    public void replaceFactorLinks(int dataPainId, List<Integer> factorIds) {
        jdbcTemplate.update("DELETE FROM public.data_pain_to_pain_factors WHERE data_pain_id = ?", dataPainId);
        for (Integer factorId : factorIds) {
            jdbcTemplate.update(
                    "INSERT INTO public.data_pain_to_pain_factors (data_pain_id, pain_factor_id) VALUES (?, ?)",
                    dataPainId, factorId);
        }
    }

    // Frontend: PT Evaluation - Pain tab (save form)
    public void replaceActivitiesImpactedLinks(int dataPainId, List<Integer> activityIds) {
        jdbcTemplate.update(
                "DELETE FROM public.data_pain_to_pain_activities_impacted WHERE data_pain_id = ?", dataPainId);
        for (Integer activityId : activityIds) {
            jdbcTemplate.update("""
                    INSERT INTO public.data_pain_to_pain_activities_impacted
                        (data_pain_id, pain_activities_impacted_id) VALUES (?, ?)
                    """, dataPainId, activityId);
        }
    }

    // --- PROM/Strength: pt-eval-prom.jsp (read) and modify_pt_prom.jsp (write) ----------------
    // Gait (non-movement-disorder) visits only: data_prom, not data_movement_disorder_prom.

    // Frontend: PT Evaluation - PROM/Strength tab (Gait visits), and the Comparison Report
    public Optional<Map<String, Object>> findPromByVisitId(int visitId) {
        return jdbcTemplate.queryForList("SELECT * FROM public.data_prom WHERE visit_id = ?", visitId)
                .stream().findFirst();
    }

    /** modify_pt_prom.jsp buckets the normal-range reference row by the patient's age at the
     * visit (less_than_age: 5, 8, 11, 14, 15, else 25) - ported as one query instead of Java
     * if/else, same bucket boundaries. Empty when data_prom_normals has no row for that bucket
     * (true of every bucket in the current seed data - flagged separately, not bent here). */
    // Frontend: PT Evaluation - PROM/Strength tab (normal-range reference row)
    public Optional<Map<String, Object>> findNormalsForVisit(int visitId) {
        return jdbcTemplate.queryForList("""
                SELECT n.*
                FROM public.data_prom_normals n, public.visits v, public.patients p
                WHERE v.id = ? AND p.id = v.pt_id
                  AND n.less_than_age = (
                    CASE
                        WHEN (v.date - p.dob) / 365.0 < 5.0 THEN 5
                        WHEN (v.date - p.dob) / 365.0 < 8.0 THEN 8
                        WHEN (v.date - p.dob) / 365.0 < 11.0 THEN 11
                        WHEN (v.date - p.dob) / 365.0 < 14.0 THEN 14
                        WHEN (v.date - p.dob) / 365.0 < 15.0 THEN 15
                        ELSE 25
                    END)
                """, visitId).stream().findFirst();
    }

    // Frontend: PT Evaluation - PROM/Strength tab (Create New Record)
    public int insertBlank(int visitId) {
        return jdbcTemplate.queryForObject(
                "INSERT INTO public.data_prom (visit_id) VALUES (?) RETURNING id", Integer.class, visitId);
    }

    /** Column names come only from this port's own fixed PROM field list, never from request
     * data, so building the SET clause from the map's keys is safe - a ~130-column fixed-arity
     * method would be far less maintainable than this port's actual legacy field count. */
    // Frontend: PT Evaluation - PROM/Strength tab (save form)
    public void update(int dataPromId, Map<String, Object> columns) {
        updateColumns("public.data_prom", dataPromId, columns);
    }

    // --- Phase 4: pt-sections.jsp's mainPromTable swap for movement-disorder visits -------------

    // Frontend: PT Evaluation - PROM/Strength tab (movement-disorder visits)
    public Optional<Map<String, Object>> findMdByVisitId(int visitId) {
        return jdbcTemplate.queryForList("SELECT * FROM public.data_movement_disorder_prom WHERE visit_id = ?",
                visitId).stream().findFirst();
    }

    // Frontend: PT Evaluation - PROM/Strength tab (movement-disorder visits, Create New Record)
    public int insertBlankMd(int visitId) {
        return jdbcTemplate.queryForObject(
                "INSERT INTO public.data_movement_disorder_prom (visit_id) VALUES (?) RETURNING id", Integer.class,
                visitId);
    }

    // Frontend: PT Evaluation - PROM/Strength tab (movement-disorder visits, save form)
    public void updateMd(int dataPromId, Map<String, Object> columns) {
        updateColumns("public.data_movement_disorder_prom", dataPromId, columns);
    }

    private void updateColumns(String table, int id, Map<String, Object> columns) {
        StringBuilder sql = new StringBuilder("UPDATE " + table + " SET ");
        List<Object> params = new ArrayList<>();
        boolean first = true;
        for (Map.Entry<String, Object> entry : columns.entrySet()) {
            if (!first) {
                sql.append(", ");
            }
            sql.append(entry.getKey()).append(" = ?");
            params.add(entry.getValue());
            first = false;
        }
        sql.append(" WHERE id = ?");
        params.add(id);
        jdbcTemplate.update(sql.toString(), params.toArray());
    }
}














package org.nemours.gaitlab.requests;

import java.util.Map;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;

public record SavePtEvaluationFormRequest(
        @NotBlank String revision,
        @NotNull Map<String, Object> values) {
}














package org.nemours.gaitlab.service;

import java.util.List;

/** Ports lab/reports/eval-report_combined_new.jsp (+ its _p2.jsp continuation and
 * comp_report-gmfm_gmfcs.jsp include) - the "View Printable Comparison Report" button on
 * pt.jsp. Reuses PtEvaluationTab's Section/Row/ColumnGroup shape, per the write-endpoint spec. */
public record ComparisonReport(String title, List<HeaderEntry> header, List<PtEvaluationTab.Section> sections) {

    public record HeaderEntry(String label, String value) {
    }
}









package org.nemours.gaitlab.service;

import java.time.LocalDate;
import java.time.format.DateTimeFormatter;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Optional;

import org.nemours.gaitlab.repository.PtEvaluationRepository;
import org.nemours.gaitlab.repository.VisitRepository;
import org.nemours.gaitlab.repository.VisitRepository.ComparisonHeaderRow;
import org.nemours.gaitlab.repository.VisitRepository.PtGeneralEvaluationRow;
import org.nemours.gaitlab.service.ComparisonReport.HeaderEntry;
import org.nemours.gaitlab.service.PtEvaluationService.General;
import org.nemours.gaitlab.service.PtEvaluationService.GeneralEvaluation;
import org.nemours.gaitlab.service.PtEvaluationService.GmfmModule;
import org.nemours.gaitlab.service.PromStrengthService.PromField;
import org.nemours.gaitlab.service.PtEvaluationTab.ColumnGroup;
import org.nemours.gaitlab.service.PtEvaluationTab.Row;
import org.nemours.gaitlab.service.PtEvaluationTab.Section;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.web.server.ResponseStatusException;

/**
 * Ports lab/reports/eval-report_combined_new.jsp (+ its _p2.jsp continuation and
 * comp_report-gmfm_gmfcs.jsp include) - pt.jsp's "View Printable Comparison Report" button.
 * Up to 3 visits side by side: the current one plus two legacy sources from session state
 * (VisitComparisonDatesHelper, set by a visit-picker UI this port doesn't have access to) - here
 * exposed as the compareVisitId1/compareVisitId2 query params the write-endpoint spec asked for.
 * Gait (non-movement-disorder) visits only.
 *
 * <p>Reuses the already-ported PtEvaluationService.getGeneral() for Ambulation Scale/GMFCS/FMS/
 * Total GMFM Score (all three are exactly the same formatted strings as the General tab, just
 * read once per compared visit) rather than re-deriving that logic a second time.
 */
@Service
public class ComparisonReportService {
    private final VisitRepository visits;
    private final PtEvaluationService generalService;
    private final PtEvaluationRepository repository;

    public ComparisonReportService(VisitRepository visits, PtEvaluationService generalService,
            PtEvaluationRepository repository) {
        this.visits = visits;
        this.generalService = generalService;
        this.repository = repository;
    }

    public ComparisonReport build(int patientId, int visitId, Integer compareVisitId1, Integer compareVisitId2) {
        visits.findInterpretationVisitByIdAndPatientId(visitId, patientId)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));

        List<Integer> visitIds = new ArrayList<>();
        visitIds.add(visitId);
        if (compareVisitId1 != null) {
            visitIds.add(compareVisitId1);
        }
        if (compareVisitId2 != null) {
            visitIds.add(compareVisitId2);
        }

        // p1VisitDate/p2VisitDate (eval-report_combined_new.jsp lines 138-139) plus the current
        // visit's own date - the column-group label legacy prints per compared visit, not a
        // synthetic "Visit N" placeholder.
        List<String> visitLabels = visitIds.stream()
                .map(vid -> visitDateLabel(vid, patientId))
                .toList();

        List<Optional<PtGeneralEvaluationRow>> rawGeneralByVisit = visitIds.stream()
                .map(vid -> visits.findPtGeneralEvaluationByVisitIdAndPatientId(vid, patientId))
                .toList();
        // getGeneral() 404s on a compare visit that doesn't belong to this patient - the same
        // boundary this port enforces everywhere else, tighter than legacy's MRN-based match.
        List<GeneralEvaluation> generalByVisit = visitIds.stream()
                .map(vid -> generalService.getGeneral(patientId, vid))
                .toList();
        List<Optional<Map<String, Object>>> promByVisit = visitIds.stream()
                .map(repository::findPromByVisitId)
                .toList();
        List<List<Map<String, Object>>> mcByVisit = visitIds.stream()
                .map(repository::findMotorControlByVisitId)
                .toList();
        List<List<Map<String, Object>>> toneByVisit = visitIds.stream()
                .map(repository::findToneByVisitId)
                .toList();

        List<Integer> promIdx = indicesWithData(promByVisit, Optional::isPresent);
        List<Integer> mcIdx = indicesWithData(mcByVisit, list -> !list.isEmpty());
        List<Integer> toneIdx = indicesWithData(toneByVisit, list -> !list.isEmpty());
        List<Integer> generalIdx = indicesWithData(rawGeneralByVisit, Optional::isPresent);

        List<Section> sections = new ArrayList<>();
        if (!promIdx.isEmpty()) {
            sections.add(promOrStrengthSection("Passive ROM", promByVisit, promIdx, visitLabels, false));
            List<Integer> strengthIdx = promIdx; // same visits; a visit with any PROM row has the strength columns too
            sections.add(promOrStrengthSection("Strength", promByVisit, strengthIdx, visitLabels, true));
            sections.add(legLengthSection(promByVisit, promIdx, visitLabels));
            sections.add(qAngleSection(promByVisit, promIdx, visitLabels));
            sections.add(popAngleSection(promByVisit, promIdx, visitLabels));
        }
        if (!mcIdx.isEmpty()) {
            sections.add(motorControlOrToneSection("Motor Control", mcByVisit, mcIdx, visitLabels));
        }
        if (!toneIdx.isEmpty()) {
            sections.add(motorControlOrToneSection("Tone", toneByVisit, toneIdx, visitLabels));
        }
        if (!generalIdx.isEmpty()) {
            sections.add(gmfmDSection(rawGeneralByVisit, generalIdx, visitLabels));
            Section mobility = mobilityScalesSection(generalByVisit, generalIdx, visitLabels);
            if (mobility != null) {
                sections.add(mobility);
            }
        }
        if (!rawGeneralByVisit.get(0).isEmpty()) {
            sections.add(otherGmfmModulesSection(rawGeneralByVisit.get(0).get()));
        }
        sections.add(notesSection(generalByVisit.get(0).general(), rawGeneralByVisit.get(0).isPresent()));

        return new ComparisonReport("Physical Therapy Evaluation - Comparison Report",
                header(patientId, visitId, generalByVisit.get(0).general()), sections);
    }

    /** The exact "M/d/yy"-style text legacy prints for a visit date (DoDateTime(date,
     * DateFormat.SHORT, null)) - falls back to the visit id if the visit somehow has no date. */
    private String visitDateLabel(int visitId, int patientId) {
        return visits.findVideoVisitDateByIdAndPatientId(visitId, patientId)
                .map(VisitRepository.VideoVisitDate::getVisitDate)
                .filter(java.util.Objects::nonNull)
                .map(text -> formatShortDate(LocalDate.parse(text)))
                .orElse("Visit " + visitId);
    }

    private static final DateTimeFormatter SHORT_DATE = DateTimeFormatter.ofPattern("M/d/yy");

    private static String formatShortDate(LocalDate date) {
        return date == null ? null : date.format(SHORT_DATE);
    }

    private <T> List<Integer> indicesWithData(List<T> perVisit, java.util.function.Predicate<T> hasData) {
        List<Integer> indices = new ArrayList<>();
        for (int i = 0; i < perVisit.size(); i++) {
            if (hasData.test(perVisit.get(i))) {
                indices.add(i);
            }
        }
        return indices;
    }

    /** Matches eval-report_combined_new.jsp's header table (lines ~480-521) field-for-field:
     * Name, then MRN/Height/Date of Eval, DOB/Weight/Doctor(interp), Age at Eval/Gender/Therapist
     * - read left-to-right, top-to-bottom. */
    private List<ComparisonReport.HeaderEntry> header(int patientId, int visitId, General general) {
        ComparisonHeaderRow row = visits.findComparisonHeaderByVisitIdAndPatientId(visitId, patientId)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));
        List<HeaderEntry> header = new ArrayList<>();
        header.add(new HeaderEntry("Name", join(row.getFirstName(), row.getLastName())));
        header.add(new HeaderEntry("MRN", row.getMrn()));
        header.add(new HeaderEntry("Height (cm)", numStr(general.heightCm())));
        header.add(new HeaderEntry("Date of Eval", formatShortDate(row.getVisitDate())));
        header.add(new HeaderEntry("Date of Birth", formatShortDate(row.getDob())));
        header.add(new HeaderEntry("Weight (kg)", numStr(general.weightKg())));
        header.add(new HeaderEntry("Doctor (interp)",
                join(row.getInterpretingDoctorFirstName(), row.getInterpretingDoctorLastName())));
        header.add(new HeaderEntry("Age at Eval", ageAtEval(row.getDob(), row.getVisitDate())));
        header.add(new HeaderEntry("Gender", row.getGender()));
        header.add(new HeaderEntry("Therapist", join(row.getTherapistFirstName(), row.getTherapistLastName())));
        return header;
    }

    /** Ported verbatim from the legacy page's own client-side age formula (not a true elapsed-time
     * calculation): age = (visitYear - dobYear) + (monthDiff/12) + (dayDiff/365), to 2 decimals. */
    private static String ageAtEval(LocalDate dob, LocalDate visitDate) {
        if (dob == null || visitDate == null) {
            return null;
        }
        double age = (visitDate.getYear() - dob.getYear())
                + ((visitDate.getMonthValue() - dob.getMonthValue()) / 12.0)
                + ((visitDate.getDayOfMonth() - dob.getDayOfMonth()) / 365.0);
        return String.format("%.2f", age);
    }

    private static String join(String first, String last) {
        String combined = ((first == null ? "" : first) + " " + (last == null ? "" : last)).trim();
        return combined.isEmpty() ? null : combined;
    }

    private static String numStr(Float value) {
        return value == null ? null : String.valueOf(value);
    }

    /** "999" -> "WNL", matching every cell in eval-report_combined_new.jsp's PROM/Strength
     * tables - this report never shows the interp code the way the PROM/Strength tab view does,
     * just the raw value. */
    private static String wnlCell(Object raw) {
        if (raw == null) {
            return null;
        }
        return "999".equals(raw.toString()) ? "WNL" : raw.toString();
    }

    private Section promOrStrengthSection(String title, List<Optional<Map<String, Object>>> promByVisit,
            List<Integer> visitIdx, List<String> visitLabels, boolean strength) {
        List<ColumnGroup> columnGroups = new ArrayList<>();
        List<String> columns = new ArrayList<>();
        columnGroups.add(new ColumnGroup("", 1));
        columns.add("");
        for (int i : visitIdx) {
            columnGroups.add(new ColumnGroup(visitLabels.get(i), 2));
            columns.add("Right");
            columns.add("Left");
        }

        List<Row> rows = new ArrayList<>();
        for (PromField field : PromStrengthService.PROM_FIELDS) {
            if (strength && !field.hasStrength()) {
                continue;
            }
            String suffix = strength ? "_s" : "";
            List<String> cells = new ArrayList<>();
            for (int i : visitIdx) {
                Map<String, Object> row = promByVisit.get(i).orElse(Map.of());
                cells.add(wnlCell(row.get(field.prefix() + suffix + "_r")));
                cells.add(wnlCell(row.get(field.prefix() + suffix + "_l")));
            }
            rows.add(new Row(field.label(), cells));
        }
        return Section.table(title, columnGroups, columns, rows, null);
    }

    private Section legLengthSection(List<Optional<Map<String, Object>>> promByVisit, List<Integer> visitIdx,
            List<String> visitLabels) {
        List<ColumnGroup> columnGroups = new ArrayList<>();
        List<String> columns = new ArrayList<>();
        List<String> cells = new ArrayList<>();
        columnGroups.add(new ColumnGroup("", 1));
        columns.add("");
        for (int i : visitIdx) {
            columnGroups.add(new ColumnGroup(visitLabels.get(i), 2));
            columns.add("Right");
            columns.add("Left");
            Map<String, Object> row = promByVisit.get(i).orElse(Map.of());
            cells.add(wnlCell(row.get("leg_length_r")));
            cells.add(wnlCell(row.get("leg_length_l")));
        }
        return Section.table("Leg Length (cm)", columnGroups, columns,
                List.of(new Row("Leg Length (cm)", cells)), null);
    }

    /** "{value} Valgus" (positive) / "{|value|} Varus" (negative) - the Comparison Report's own
     * wording, different from the "+"/"-" sign prefix modify_pt_prom.jsp and the PROM/Strength
     * tab use for the same q_angle_r/l columns. Ported as its own distinct format, not bent to
     * match the other two. */
    private static String qAngleCell(Object raw) {
        if (raw == null) {
            return null;
        }
        if ("999".equals(raw.toString())) {
            return "WNL";
        }
        int value = Integer.parseInt(raw.toString());
        return value > 0 ? value + " Valgus" : (value * -1) + " Varus";
    }

    private Section qAngleSection(List<Optional<Map<String, Object>>> promByVisit, List<Integer> visitIdx,
            List<String> visitLabels) {
        List<ColumnGroup> columnGroups = new ArrayList<>();
        List<String> columns = new ArrayList<>();
        List<String> cells = new ArrayList<>();
        columnGroups.add(new ColumnGroup("", 1));
        columns.add("");
        for (int i : visitIdx) {
            columnGroups.add(new ColumnGroup(visitLabels.get(i), 2));
            columns.add("Right");
            columns.add("Left");
            Map<String, Object> row = promByVisit.get(i).orElse(Map.of());
            cells.add(qAngleCell(row.get("q_angle_r")));
            cells.add(qAngleCell(row.get("q_angle_l")));
        }
        return Section.table("Knee Varus/Valgus", columnGroups, columns,
                List.of(new Row("Knee Varus/Valgus", cells)), null);
    }

    private Section popAngleSection(List<Optional<Map<String, Object>>> promByVisit, List<Integer> visitIdx,
            List<String> visitLabels) {
        List<ColumnGroup> columnGroups = new ArrayList<>();
        List<String> columns = new ArrayList<>();
        List<String> cells = new ArrayList<>();
        columnGroups.add(new ColumnGroup("", 1));
        columns.add("");
        for (int i : visitIdx) {
            columnGroups.add(new ColumnGroup(visitLabels.get(i), 2));
            columns.add("Right");
            columns.add("Left");
            Map<String, Object> row = promByVisit.get(i).orElse(Map.of());
            cells.add(rangeCell(row.get("pop_r_low"), row.get("pop_r_high")));
            cells.add(rangeCell(row.get("pop_l_low"), row.get("pop_l_high")));
        }
        return Section.table("Pop angle", columnGroups, columns, List.of(new Row("Pop angle", cells)), null);
    }

    private static String rangeCell(Object low, Object high) {
        if (low == null && high == null) {
            return null;
        }
        return wnlCell(low) + " -- " + wnlCell(high);
    }

    private Section motorControlOrToneSection(String title, List<List<Map<String, Object>>> dataByVisit,
            List<Integer> visitIdx, List<String> visitLabels) {
        List<ColumnGroup> columnGroups = new ArrayList<>();
        List<String> columns = new ArrayList<>();
        columnGroups.add(new ColumnGroup("", 1));
        columns.add("");
        for (int i : visitIdx) {
            columnGroups.add(new ColumnGroup(visitLabels.get(i), 2));
            columns.add("Right");
            columns.add("Left");
        }

        // The legacy comparison report, unlike the Motor Control/Tone tab views, does not grey
        // or hide the score for "*"-prefixed (interpretation-only) tests - it just prints
        // whatever's in score_r/score_l for every row, same as any other test.
        List<Map<String, Object>> primary = dataByVisit.get(visitIdx.get(0));
        List<Row> rows = new ArrayList<>();
        for (int rowIndex = 0; rowIndex < primary.size(); rowIndex++) {
            String test = (String) primary.get(rowIndex).get("test");
            List<String> cells = new ArrayList<>();
            for (int i : visitIdx) {
                List<Map<String, Object>> rowsForVisit = dataByVisit.get(i);
                Map<String, Object> row = rowIndex < rowsForVisit.size() ? rowsForVisit.get(rowIndex) : Map.of();
                cells.add(str(row.get("score_r")));
                cells.add(str(row.get("score_l")));
            }
            rows.add(new Row(test, cells));
        }
        return Section.table(title, columnGroups, columns, rows, null);
    }

    private static String str(Object value) {
        return value == null ? null : value.toString();
    }

    private Section gmfmDSection(List<Optional<PtGeneralEvaluationRow>> rawGeneralByVisit, List<Integer> visitIdx,
            List<String> visitLabels) {
        List<ColumnGroup> columnGroups = new ArrayList<>();
        List<String> columns = new ArrayList<>();
        columnGroups.add(new ColumnGroup("", 1));
        columns.add("");
        for (int i : visitIdx) {
            columnGroups.add(new ColumnGroup(visitLabels.get(i), 2));
            columns.add("Right");
            columns.add("Left");
        }

        record Task(String label, String rightGetter, String leftGetter) {
        }
        // Mirrors printFunctionalStrengthGMFMRow()'s own task list and labels exactly.
        List<Row> rows = new ArrayList<>();
        rows.add(gmfmDRow("Pull to stand (52)", rawGeneralByVisit, visitIdx, PtGeneralEvaluationRow::getPullStand, null));
        rows.add(gmfmDRow("Independent Standing - Arms Free (3 sec) (53)", rawGeneralByVisit, visitIdx,
                PtGeneralEvaluationRow::getIndepStand3, null));
        rows.add(gmfmDRow("Independent Standing - Arms Free (20 sec) (53)", rawGeneralByVisit, visitIdx,
                PtGeneralEvaluationRow::getIndepStand20, null));
        rows.add(gmfmDRow("One leg stance hand held - Right (54,55)", rawGeneralByVisit, visitIdx,
                PtGeneralEvaluationRow::getOneLegHandR, PtGeneralEvaluationRow::getOneLegHandL));
        rows.add(gmfmDRow("One leg stance (57,58)", rawGeneralByVisit, visitIdx, PtGeneralEvaluationRow::getOneLegR,
                PtGeneralEvaluationRow::getOneLegL));
        rows.add(gmfmDRow("Short sit to stand (59)", rawGeneralByVisit, visitIdx,
                PtGeneralEvaluationRow::getShortSitStand, null));
        rows.add(gmfmDRow("High Kneel to stand (60,61)", rawGeneralByVisit, visitIdx,
                PtGeneralEvaluationRow::getHighKneeStandR, PtGeneralEvaluationRow::getHighKneeStandL));
        rows.add(gmfmDRow("Lowers to floor (62)", rawGeneralByVisit, visitIdx, PtGeneralEvaluationRow::getLowerFloor,
                null));
        rows.add(gmfmDRow("Squats (63)", rawGeneralByVisit, visitIdx, PtGeneralEvaluationRow::getSquats, null));
        rows.add(gmfmDRow("Standing-picks pen from floor (64)", rawGeneralByVisit, visitIdx,
                PtGeneralEvaluationRow::getStandPickPen, null));
        return Section.table("Functional Strength - GMFM-D", columnGroups, columns, rows, null);
    }

    private Row gmfmDRow(String label, List<Optional<PtGeneralEvaluationRow>> rawGeneralByVisit,
            List<Integer> visitIdx, java.util.function.Function<PtGeneralEvaluationRow, Integer> right,
            java.util.function.Function<PtGeneralEvaluationRow, Integer> left) {
        List<String> cells = new ArrayList<>();
        for (int i : visitIdx) {
            PtGeneralEvaluationRow row = rawGeneralByVisit.get(i).orElse(null);
            Integer rightValue = row == null ? null : right.apply(row);
            cells.add(rightValue == null ? "--" : String.valueOf(rightValue));
            if (left != null) {
                Integer leftValue = row == null ? null : left.apply(row);
                cells.add(leftValue == null ? "--" : String.valueOf(leftValue));
            } else {
                cells.add(null);
            }
        }
        return new Row(label, cells);
    }

    private Section mobilityScalesSection(List<GeneralEvaluation> generalByVisit, List<Integer> visitIdx,
            List<String> visitLabels) {
        List<String> columns = new ArrayList<>();
        columns.add("");
        visitIdx.forEach(i -> columns.add(visitLabels.get(i)));
        List<Row> rows = new ArrayList<>();
        rows.add(scaleRow("Ambulation Scale", generalByVisit, visitIdx, General::ambulationScale));

        boolean anyGmfcs = visitIdx.stream().anyMatch(i -> generalByVisit.get(i).general().gmfcs() != null);
        if (anyGmfcs) {
            rows.add(scaleRowWithDash("GMFCS", generalByVisit, visitIdx, General::gmfcs));
        }
        addIfAny(rows, "FMS (5m)", generalByVisit, visitIdx, General::fms5m);
        addIfAny(rows, "FMS (50m)", generalByVisit, visitIdx, General::fms50m);
        addIfAny(rows, "FMS (500m)", generalByVisit, visitIdx, General::fms500m);

        List<String> totalGmfm = new ArrayList<>();
        for (int i : visitIdx) {
            GmfmModule moduleD = generalByVisit.get(i).gmfmModules().stream()
                    .filter(m -> "D".equals(m.module())).findFirst().orElse(null);
            int score = moduleD == null || moduleD.score() == null ? 0 : moduleD.score();
            totalGmfm.add(score > 0 ? score + " / 39" : "N/A");
        }
        rows.add(new Row("Total GMFM Score", totalGmfm));

        return Section.table("Mobility Scales", null, columns, rows, null);
    }

    private void addIfAny(List<Row> rows, String label, List<GeneralEvaluation> generalByVisit,
            List<Integer> visitIdx, java.util.function.Function<General, String> extractor) {
        boolean any = visitIdx.stream().anyMatch(i -> extractor.apply(generalByVisit.get(i).general()) != null);
        if (any) {
            rows.add(scaleRow(label, generalByVisit, visitIdx, extractor));
        }
    }

    private Row scaleRow(String label, List<GeneralEvaluation> generalByVisit, List<Integer> visitIdx,
            java.util.function.Function<General, String> extractor) {
        List<String> cells = visitIdx.stream().map(i -> extractor.apply(generalByVisit.get(i).general())).toList();
        return new Row(label, cells);
    }

    /** Same as scaleRow, but a visit with a null value (not CP, or CP with nothing resolved)
     * shows "--" rather than a blank cell, matching comp_report-gmfm_gmfcs.jsp's own GMFCS row -
     * every other scale row leaves a missing visit blank instead. */
    private Row scaleRowWithDash(String label, List<GeneralEvaluation> generalByVisit, List<Integer> visitIdx,
            java.util.function.Function<General, String> extractor) {
        List<String> cells = new ArrayList<>();
        for (int i : visitIdx) {
            String value = extractor.apply(generalByVisit.get(i).general());
            cells.add(value == null ? "--" : value);
        }
        return new Row(label, cells);
    }

    /** Current visit only - comp_report-gmfm_gmfcs.jsp never compares these across visits, just
     * prints the current one's Module A/B/C/E (checking only for null, not the "0 means no
     * entry" rule the General tab's own GmfmModule scoring applies - a real difference between
     * the two legacy pages, not a bug to normalize away). */
    private Section otherGmfmModulesSection(PtGeneralEvaluationRow row) {
        List<String> lines = new ArrayList<>();
        if (row.getGmfmModA() != null) {
            lines.add("Module A: " + row.getGmfmModA());
        }
        if (row.getGmfmModB() != null) {
            lines.add("Module B: " + row.getGmfmModB());
        }
        if (row.getGmfmModC() != null) {
            lines.add("Module C: " + row.getGmfmModC());
        }
        if (row.getGmfmModE() != null) {
            lines.add("Module E: " + row.getGmfmModE());
        }
        return Section.list("Other GMFM Modules", lines);
    }

    /** Current visit only, matching the legacy "Notes:" footer. */
    private Section notesSection(General general, boolean hasGeneralData) {
        String text = hasGeneralData ? general.comments() : null;
        return Section.text("Notes", hasGeneralData ? text : "No GMFM Data Recorded.");
    }
}








package org.nemours.gaitlab.service;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Optional;

import org.nemours.gaitlab.repository.PtEvaluationRepository;
import org.nemours.gaitlab.service.PtEvaluationForm.Field;
import org.nemours.gaitlab.service.PtEvaluationForm.FormSection;
import org.nemours.gaitlab.service.PtEvaluationForm.Option;
import org.nemours.gaitlab.service.PtEvaluationTab.Row;
import org.nemours.gaitlab.service.PtEvaluationTab.Section;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.server.ResponseStatusException;

/** Ports pt-eval-dystonia-scale.jsp/modify_pt_dystonia_scale.jsp - the Dystonia Scale tab,
 * movement-disorder visits only. One "Score" column (not Right/Left) - Eyes/Mouth/Neck/Trunk are
 * unilateral (one value per row); UE/LE are bilateral and show as "Right: ...\nLeft: ..." in that
 * SAME single cell (pt-eval-dystonia-scale.jsp:143-160 renders both as separate &lt;p&gt;s inside
 * one &lt;TD&gt;, not two columns). Row label is "{item}: {longdesc}". */
@Service
public class DystoniaService {
    private static final int UE_VALUE = 5;
    private static final int LE_VALUE = 6;

    private final PtEvaluationRepository repository;
    private final PtEvaluationRevision revision;

    public DystoniaService(PtEvaluationRepository repository, PtEvaluationRevision revision) {
        this.repository = repository;
        this.revision = revision;
    }

    public PtEvaluationTab build(int patientId, int visitId, String tab) {
        Optional<Map<String, Object>> scale = repository.findDystoniaScaleByVisitId(visitId);
        if (scale.isEmpty()) {
            return PtEvaluationTab.empty(patientId, visitId, tab, "No Dystonia scores entered.");
        }

        List<Map<String, Object>> items = repository.findDystoniaItems();
        List<Row> rows = new ArrayList<>();
        for (Map<String, Object> item : items) {
            int itemValue = (int) item.get("value");
            String label = rowLabel(item);
            String score;
            if (itemValue == UE_VALUE || itemValue == LE_VALUE) {
                Integer right = column(scale.get(), itemValue, "r");
                Integer left = column(scale.get(), itemValue, "l");
                score = "Right: " + cell(right) + "\nLeft: " + cell(left);
            } else {
                Integer value = column(scale.get(), itemValue, null);
                score = cell(value);
            }
            rows.add(new Row(label, java.util.Collections.singletonList(score)));
        }
        Section section = Section.table(null, null, List.of("", "Score"), rows, null);
        return PtEvaluationTab.of(patientId, visitId, tab, List.of(section));
    }

    private static String rowLabel(Map<String, Object> item) {
        return item.get("item") + ": " + item.get("longdesc");
    }

    private String cell(Integer selectionId) {
        return selectionId == null ? "--" : formatSelection(selectionId);
    }

    private String formatSelection(int selectionId) {
        return repository.findDystoniaSelectionById(selectionId).map(DystoniaService::selectionLabel).orElse("--");
    }

    /** "{value}={selection}", "*" for a null value ("Unable to assess...") - matches
     * modify_pt_dystonia_scale.jsp's own `hs.getValue() == null ? "*" : hs.getValue()` exactly. */
    private static String selectionLabel(Map<String, Object> selection) {
        Object value = selection.get("value");
        return (value == null ? "*" : value) + "=" + selection.get("selection");
    }

    /** item value 1-4 (Eyes/Mouth/Neck/Trunk) -> eyes/mouth/neck/trunk (side is ignored); 5 (UE)
     * -> uer/uel; 6 (LE) -> ler/lel. Matches DystoniaScale's own column layout. */
    private static Integer column(Map<String, Object> scale, int itemValue, String side) {
        String column = switch (itemValue) {
            case 1 -> "eyes";
            case 2 -> "mouth";
            case 3 -> "neck";
            case 4 -> "trunk";
            case 5 -> "r".equals(side) ? "uer" : "uel";
            case 6 -> "r".equals(side) ? "ler" : "lel";
            default -> throw new IllegalArgumentException("Unknown dystonia item value " + itemValue);
        };
        return (Integer) scale.get(column);
    }

    public PtEvaluationForm buildForm(int visitId) {
        Map<String, Object> scale = repository.findDystoniaScaleByVisitId(visitId)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND,
                        "No Dystonia Scale record exists yet for this visit."));

        List<Map<String, Object>> items = repository.findDystoniaItems();
        List<Field> fields = new ArrayList<>();
        for (Map<String, Object> item : items) {
            int itemId = (int) item.get("id");
            int itemValue = (int) item.get("value");
            String label = rowLabel(item);
            List<Option> rawOptions = toOptions(repository.findDystoniaSelectionsForItem(itemId));
            if (itemValue == UE_VALUE || itemValue == LE_VALUE) {
                Integer right = column(scale, itemValue, "r");
                Integer left = column(scale, itemValue, "l");
                fields.add(Field.radio(columnName(itemValue, "r"), label + " Right",
                        right == null ? null : String.valueOf(right), optionsIncluding(rawOptions, right)));
                fields.add(Field.radio(columnName(itemValue, "l"), label + " Left",
                        left == null ? null : String.valueOf(left), optionsIncluding(rawOptions, left)));
            } else {
                Integer value = column(scale, itemValue, null);
                fields.add(Field.radio(columnName(itemValue, null), label,
                        value == null ? null : String.valueOf(value), optionsIncluding(rawOptions, value)));
            }
        }

        return new PtEvaluationForm("dystonia-scale", "Edit Dystonia Scale Values", revision.of(scale),
                List.of(FormSection.stack(null, fields)));
    }

    private static List<Option> toOptions(List<Map<String, Object>> selections) {
        return selections.stream()
                .map(selection -> new Option(String.valueOf(selection.get("id")), selectionLabel(selection)))
                .toList();
    }

    private static String columnName(int itemValue, String side) {
        return switch (itemValue) {
            case 1 -> "eyes";
            case 2 -> "mouth";
            case 3 -> "neck";
            case 4 -> "trunk";
            case 5 -> "r".equals(side) ? "uer" : "uel";
            case 6 -> "r".equals(side) ? "ler" : "lel";
            default -> throw new IllegalArgumentException("Unknown dystonia item value " + itemValue);
        };
    }

    private List<Option> optionsIncluding(List<Option> selections, Integer currentId) {
        List<Option> options = new ArrayList<>(selections);
        boolean hasCurrent = currentId == null
                || options.stream().anyMatch(option -> option.value().equals(String.valueOf(currentId)));
        if (!hasCurrent) {
            repository.findDystoniaSelectionById(currentId).ifPresent(selection -> options.add(0,
                    new Option(String.valueOf(currentId), selectionLabel(selection))));
        }
        return options;
    }

    @Transactional
    public PtEvaluationTab save(int patientId, int visitId, String submittedRevision, Map<String, Object> values) {
        Map<String, Object> scale = repository.findDystoniaScaleByVisitId(visitId)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND,
                        "No Dystonia Scale record exists yet for this visit."));

        if (!revision.of(scale).equals(submittedRevision)) {
            throw new ResponseStatusException(HttpStatus.CONFLICT,
                    "This data has changed since you loaded it. Reload and try again.");
        }

        List<Map<String, Object>> items = repository.findDystoniaItems();
        Map<String, String> errors = new java.util.LinkedHashMap<>();
        Map<String, Integer> resolved = new java.util.LinkedHashMap<>();
        for (Map<String, Object> item : items) {
            int itemId = (int) item.get("id");
            int itemValue = (int) item.get("value");
            if (itemValue == UE_VALUE || itemValue == LE_VALUE) {
                for (String side : List.of("r", "l")) {
                    String key = columnName(itemValue, side);
                    Integer current = column(scale, itemValue, side);
                    resolved.put(key, resolveLookupId(values, key, current,
                            id -> repository.existsDystoniaSelectionForItem(itemId, id), errors));
                }
            } else {
                String key = columnName(itemValue, null);
                Integer current = column(scale, itemValue, null);
                resolved.put(key, resolveLookupId(values, key, current,
                        id -> repository.existsDystoniaSelectionForItem(itemId, id), errors));
            }
        }

        if (!errors.isEmpty()) {
            throw new PtEvaluationValidationException("Some Dystonia Scale values are invalid.", errors);
        }

        repository.updateDystoniaScale((int) scale.get("id"), resolved.get("eyes"), resolved.get("mouth"),
                resolved.get("neck"), resolved.get("trunk"), resolved.get("uel"), resolved.get("uer"),
                resolved.get("lel"), resolved.get("ler"));

        return build(patientId, visitId, "dystonia-scale");
    }

    private interface IdExists {
        boolean test(int id);
    }

    private static Integer resolveLookupId(Map<String, Object> values, String key, Integer currentValue,
            IdExists exists, Map<String, String> errors) {
        if (!values.containsKey(key)) {
            return currentValue;
        }
        Object raw = values.get(key);
        String submitted = raw == null ? null : raw.toString();
        if (submitted == null || submitted.isEmpty()) {
            return null;
        }
        Integer resolvedValue;
        try {
            resolvedValue = Integer.parseInt(submitted);
        } catch (NumberFormatException e) {
            errors.put(key, "Must be a whole number.");
            return currentValue;
        }
        if (!exists.test(resolvedValue)) {
            errors.put(key, "Unknown value.");
            return currentValue;
        }
        return resolvedValue;
    }

    public PtEvaluationTab createRecord(int patientId, int visitId) {
        if (repository.findDystoniaScaleByVisitId(visitId).isPresent()) {
            throw new ResponseStatusException(HttpStatus.CONFLICT, "Dystonia Scale data already exists for this visit.");
        }
        repository.insertBlankDystoniaScale(visitId);
        return build(patientId, visitId, "dystonia-scale");
    }
}











package org.nemours.gaitlab.service;

import java.time.LocalDate;
import java.time.temporal.ChronoUnit;
import java.util.List;
import java.util.Map;

import org.nemours.gaitlab.repository.PtEvaluationRepository;
import org.nemours.gaitlab.repository.VisitRepository;
import org.nemours.gaitlab.repository.VisitRepository.PtGeneralMdEvaluationRow;
import org.nemours.gaitlab.service.PtEvaluationForm.Field;
import org.nemours.gaitlab.service.PtEvaluationForm.FormSection;
import org.nemours.gaitlab.service.PtEvaluationForm.Option;
import org.nemours.gaitlab.service.PtEvaluationTab.Row;
import org.nemours.gaitlab.service.PtEvaluationTab.Section;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.server.ResponseStatusException;

/**
 * Ports lab/data/pt.jsp's isMovementDisorder branch (key "general-md" in the frontend's tab
 * contract) - the "General" tab for movement-disorder visits, backed by
 * data_pt_movement_disorder instead of data_pt. No GMFM-D task table and no Other GMFM Modules
 * section - data_pt_movement_disorder has no such columns, and pt.jsp's own MD branch never
 * renders that part of the page. GMFCS here is a plain lookup label like Ambulation Scale/FMS,
 * not the CP-diagnosis-gated calculated-vs-manual logic PtEvaluationService uses for Gait - that
 * calculation depends on GMFM-D task scores, which don't exist on this table, so there is nothing
 * to calculate from; this is a deliberate simplification, not an oversight.
 *
 * <p>Uses the same generic PtEvaluationTab (label/value row) contract as every other tab, not a
 * bespoke shape - height/weight/foot measurements, Ambulation Scale, GMFCS, FMS (5m/50m/500m),
 * MACS, CFCS and Comments all render as one row apiece, matching pt.jsp's own row order exactly
 * (lines 391-527) - height/weight/foot rows are NOT gated on isMovementDisorder in that JSP, they
 * just read from the same visits columns regardless of which table the rest of the page reads.
 */
@Service
public class GeneralMdService {
    static final String EMPTY_MESSAGE = "No general data has been entered yet for this visit.";

    private final VisitRepository visits;
    private final PtEvaluationRepository repository;
    private final PtEvaluationRevision revision;

    public GeneralMdService(VisitRepository visits, PtEvaluationRepository repository,
            PtEvaluationRevision revision) {
        this.visits = visits;
        this.repository = repository;
        this.revision = revision;
    }

    public PtEvaluationTab getGeneral(int patientId, int visitId, String tab) {
        visits.findInterpretationVisitByIdAndPatientId(visitId, patientId)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));

        return visits.findPtGeneralMdEvaluationByVisitIdAndPatientId(visitId, patientId)
                .map(row -> PtEvaluationTab.of(patientId, visitId, tab, List.of(buildSection(row))))
                .orElseGet(() -> PtEvaluationTab.empty(patientId, visitId, tab, EMPTY_MESSAGE));
    }

    private Section buildSection(PtGeneralMdEvaluationRow row) {
        List<Row> rows = new java.util.ArrayList<>(List.of(
                cellRow("Height (cm)", valueWithZ(row.getHeightCm(),
                        heightZ(row.getHeightCm(), row.getGender(), row.getVisitDate(), row.getDob()))),
                cellRow("Weight (kg)", valueWithZ(row.getWeightKg(),
                        weightZ(row.getWeightKg(), row.getGender(), row.getVisitDate(), row.getDob()))),
                cellRow("Right Foot Length (cm)", numStr(row.getRightFootLengthCm())),
                cellRow("Left Foot Length (cm)", numStr(row.getLeftFootLengthCm())),
                cellRow("Right Foot Width (cm)", numStr(row.getRightFootWidthCm())),
                cellRow("Left Foot Width (cm)", numStr(row.getLeftFootWidthCm())),
                cellRow("Ambulation Scale", labelDash(
                        row.getAmbScaleId() == null ? null : String.valueOf(row.getAmbScaleId()),
                        row.getAmbulationScale()))));
        // pt.jsp wraps the entire GMFCS <TR> in the same isCP(gen_diag)||isCP(diagnosis) check for
        // BOTH the Gait and movement-disorder branches (lines 454-456) - this row isn't
        // unconditional for MD the way Ambulation Scale/FMS/MACS/CFCS are.
        if (isCP(row.getGenDiag()) || isCP(row.getIcd9())) {
            rows.add(cellRow("GMFCS", labelDash(row.getGmfcsLabel(), row.getGmfcsDesc())));
        }
        rows.add(cellRow("FMS (5m)", labelDash(row.getFms5m(), row.getFms5mDesc())));
        rows.add(cellRow("FMS (50m)", labelDash(row.getFms50m(), row.getFms50mDesc())));
        rows.add(cellRow("FMS (500m)", labelDash(row.getFms500m(), row.getFms500mDesc())));
        rows.add(cellRow("MACS", labelDash(row.getMacsLabel(), row.getMacsDesc())));
        rows.add(cellRow("CFCS", labelDash(row.getCfcsLabel(), row.getCfcsDesc())));
        rows.add(cellRow("Comments", row.getComments()));
        return Section.table("General and Marker Information", null, List.of("", ""), rows, null);
    }

    /** Same CP-diagnosis check as PtEvaluationService's own isCP - duplicated rather than shared
     * since pt.jsp itself inlines this exact condition separately in both branches. */
    private static boolean isCP(Integer icd9Id) {
        return icd9Id != null && ((icd9Id >= 13 && icd9Id <= 17) || icd9Id == 119 || icd9Id == 142);
    }

    /** A single-cell Row - List.of(cell) would NPE whenever cell is null (height/weight/foot
     * measurements and comments regularly are), so this always goes through
     * Collections.singletonList, which tolerates it. */
    private static Row cellRow(String label, String cell) {
        return new Row(label, java.util.Collections.singletonList(cell));
    }

    /** "{value} (z = {score})", matching pt.jsp's height/weight rows where the z-score sits in
     * the same cell right after the raw value - null value means no row content at all. */
    private static String valueWithZ(Float value, String z) {
        if (value == null) {
            return null;
        }
        return numStr(value) + (z == null ? "" : " " + z);
    }

    private static String labelDash(String value, String description) {
        return (value == null ? "" : value) + " -- " + (description == null ? "" : description);
    }

    private static final java.text.DecimalFormat Z_FORMAT_PATTERN = new java.text.DecimalFormat("0.0");

    private static String zFormat(double value) {
        synchronized (Z_FORMAT_PATTERN) {
            return Z_FORMAT_PATTERN.format(value);
        }
    }

    /** Same growth-curve formulas as PtEvaluationService.heightZ/weightZ - duplicated rather than
     * shared because pt.jsp itself duplicates this exact block verbatim into its own
     * isMovementDisorder branch instead of factoring it out. */
    private String heightZ(Float heightCm, String gender, LocalDate visitDate, LocalDate dob) {
        if (heightCm == null) {
            return null;
        }
        if (!"Male".equals(gender) && !"Female".equals(gender)) {
            return "(z = need gender)";
        }
        if (visitDate == null || dob == null) {
            return null;
        }
        double months = ageInMonths(visitDate, dob);
        double mean;
        double std;
        if ("Male".equals(gender)) {
            mean = -260.3657 + (262.8692 - (-260.3657)) / (1.0 + 0.57774 * Math.exp(-(0.0052597 * months)));
            std = 2.02485 + (9.8984 - 2.02485) / (1.0 + 7.5227 * Math.exp(-(0.0181343 * months)));
        } else {
            mean = -82.788774 + (195.26046 - (-82.798774)) / (1.0 + 0.8455536 * Math.exp(-(0.0101596 * months)));
            std = 2.40482737 + (6.81956506 - 2.40482737) / (1.0 + 15.723074 * Math.exp(-(0.0450524084 * months)));
        }
        return "(z = " + zFormat((heightCm - mean) / std) + ")";
    }

    private String weightZ(Float weightKg, String gender, LocalDate visitDate, LocalDate dob) {
        if (weightKg == null) {
            return null;
        }
        if (!"Male".equals(gender) && !"Female".equals(gender)) {
            return "(z = N/A)";
        }
        if (visitDate == null || dob == null) {
            return null;
        }
        double months = ageInMonths(visitDate, dob);
        double mean;
        double std;
        if ("Male".equals(gender)) {
            mean = 3.7 + (146.93 - 3.736) / (1.0 + 21.068 * Math.exp(-(0.013832 * months)));
            std = 0.872199 + (13.9084 - 0.872199) / (1.0 + 59.325 * Math.exp(-(0.0338 * months)));
        } else {
            mean = 6.4032907948 + (74.129378344 - 6.4032907947) / (1.0 + 19.0375155389 * Math.exp(-(0.020912892101 * months)));
            std = 1.16994385 + (12.9379166 - 1.1699438) / (1.0 + 88.52793975 * Math.exp(-(0.041292875 * months)));
        }
        return "(z = " + zFormat((weightKg - mean) / std) + ")";
    }

    private static double ageInMonths(LocalDate visitDate, LocalDate dob) {
        long days = ChronoUnit.DAYS.between(dob, visitDate);
        return Math.round(days / 365.0 * 12);
    }

    // --- Phase 4: modify_pt.jsp's isMovementDisorder edit-form branch --------------------------

    private static final List<Option> AMB_SCALE_OPTIONS = List.of(
            new Option("1", "1"), new Option("2", "2"), new Option("3", "3"),
            new Option("4", "4"), new Option("5", "5"), new Option("6", "6"));

    public PtEvaluationForm buildForm(int patientId, int visitId) {
        PtGeneralMdEvaluationRow row = visits.findPtGeneralMdEvaluationByVisitIdAndPatientId(visitId, patientId)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND,
                        "No general record exists yet for this visit."));

        Field height = Field.text("height", "Height (cm)", numStr(row.getHeightCm()));
        Field weight = Field.text("weight", "Weight (kg)", numStr(row.getWeightKg()));
        Field rFootLength = Field.text("r_foot_length", "Right Foot Length (cm)", numStr(row.getRightFootLengthCm()));
        Field lFootLength = Field.text("l_foot_length", "Left Foot Length (cm)", numStr(row.getLeftFootLengthCm()));
        Field rFootWidth = Field.text("r_foot_width", "Right Foot Width (cm)", numStr(row.getRightFootWidthCm()));
        Field lFootWidth = Field.text("l_foot_width", "Left Foot Width (cm)", numStr(row.getLeftFootWidthCm()));
        Field ambScale = new Field("amb_scale", "Ambulation Scale", "radio", idStr(row.getAmbScaleId()),
                AMB_SCALE_OPTIONS, false, null, null, null, null, null);
        Field gmfcs = Field.select("gmfcs", "GMFCS", idStr(row.getGmfcsId()), toIdLabelOptions(gmfcsListEntries()));
        Field fms5 = Field.select("fms_5", "FMS (5m)", idStr(row.getFms5Id()), toIdLabelOptions(fmsListEntries()));
        Field fms50 = Field.select("fms_50", "FMS (50m)", idStr(row.getFms50Id()), toIdLabelOptions(fmsListEntries()));
        Field fms500 = Field.select("fms_500", "FMS (500m)", idStr(row.getFms500Id()),
                toIdLabelOptions(fmsListEntries()));
        Field macs = Field.select("macs", "MACS", idStr(row.getMacsId()),
                toIdLabelOptions(repository.findActiveMacsListEntries()));
        Field cfcs = Field.select("cfcs", "CFCS", idStr(row.getCfcsId()),
                toIdLabelOptions(repository.findActiveCfcsListEntries()));
        Field comments = Field.textarea("comments", "Comments", row.getComments());

        FormSection generalSection = FormSection.stack("General and Marker Information", List.of(
                height, weight, rFootLength, lFootLength, rFootWidth, lFootWidth, ambScale, gmfcs, fms5, fms50,
                fms500, macs, cfcs, comments));

        return new PtEvaluationForm("general-md", "Modify General Data", revision.of(formSnapshot(row)),
                List.of(generalSection));
    }

    private List<Map<String, Object>> gmfcsListEntries() {
        return repository.findActiveGmfcsListEntries();
    }

    private List<Map<String, Object>> fmsListEntries() {
        return repository.findActiveFmsListEntries();
    }

    private static List<Option> toIdLabelOptions(List<Map<String, Object>> rows) {
        return rows.stream().map(row -> new Option(String.valueOf(row.get("id")), (String) row.get("label")))
                .toList();
    }

    private static String idStr(Integer value) {
        return value == null ? null : String.valueOf(value);
    }

    private static String numStr(Float value) {
        return value == null ? null : String.valueOf(value);
    }

    private static Map<String, Object> formSnapshot(PtGeneralMdEvaluationRow row) {
        Map<String, Object> snapshot = new java.util.LinkedHashMap<>();
        snapshot.put("heightCm", row.getHeightCm());
        snapshot.put("weightKg", row.getWeightKg());
        snapshot.put("rightFootLengthCm", row.getRightFootLengthCm());
        snapshot.put("leftFootLengthCm", row.getLeftFootLengthCm());
        snapshot.put("rightFootWidthCm", row.getRightFootWidthCm());
        snapshot.put("leftFootWidthCm", row.getLeftFootWidthCm());
        snapshot.put("ambScaleId", row.getAmbScaleId());
        snapshot.put("gmfcsId", row.getGmfcsId());
        snapshot.put("fms5Id", row.getFms5Id());
        snapshot.put("fms50Id", row.getFms50Id());
        snapshot.put("fms500Id", row.getFms500Id());
        snapshot.put("macsId", row.getMacsId());
        snapshot.put("cfcsId", row.getCfcsId());
        snapshot.put("comments", row.getComments());
        return snapshot;
    }

    @Transactional
    public PtEvaluationTab save(int patientId, int visitId, String submittedRevision,
            Map<String, Object> values) {
        PtGeneralMdEvaluationRow row = visits.findPtGeneralMdEvaluationByVisitIdAndPatientId(visitId, patientId)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND,
                        "No general record exists yet for this visit."));

        if (!revision.of(formSnapshot(row)).equals(submittedRevision)) {
            throw new ResponseStatusException(HttpStatus.CONFLICT,
                    "This data has changed since you loaded it. Reload and try again.");
        }

        Map<String, String> errors = new java.util.LinkedHashMap<>();
        Float height = resolveFloat(values, "height", row.getHeightCm(), errors);
        Float weight = resolveFloat(values, "weight", row.getWeightKg(), errors);
        Float rFootLength = resolveFloat(values, "r_foot_length", row.getRightFootLengthCm(), errors);
        Float lFootLength = resolveFloat(values, "l_foot_length", row.getLeftFootLengthCm(), errors);
        Float rFootWidth = resolveFloat(values, "r_foot_width", row.getRightFootWidthCm(), errors);
        Float lFootWidth = resolveFloat(values, "l_foot_width", row.getLeftFootWidthCm(), errors);
        Integer ambScale = resolveFromSet(values, "amb_scale", row.getAmbScaleId(),
                java.util.Set.of(1, 2, 3, 4, 5, 6), errors);
        Integer gmfcs = resolveLookupId(values, "gmfcs", row.getGmfcsId(), repository::existsGmfcsListEntry,
                errors);
        Integer fms5 = resolveLookupId(values, "fms_5", row.getFms5Id(), repository::existsFmsListEntry,
                errors);
        Integer fms50 = resolveLookupId(values, "fms_50", row.getFms50Id(), repository::existsFmsListEntry,
                errors);
        Integer fms500 = resolveLookupId(values, "fms_500", row.getFms500Id(), repository::existsFmsListEntry,
                errors);
        Integer macs = resolveLookupId(values, "macs", row.getMacsId(), repository::existsMacsListEntry, errors);
        Integer cfcs = resolveLookupId(values, "cfcs", row.getCfcsId(), repository::existsCfcsListEntry, errors);
        String comments = resolveText(values, "comments", row.getComments());

        if (!errors.isEmpty()) {
            throw new PtEvaluationValidationException("Some general fields are invalid.", errors);
        }

        repository.updateVisitMeasurements(visitId, height, weight, rFootLength, lFootLength, rFootWidth,
                lFootWidth);
        repository.updateDataPtMovementDisorder(row.getDataPtMdId(), ambScale, gmfcs, fms5, fms50, fms500, macs,
                cfcs, comments);

        return getGeneral(patientId, visitId, "general-md");
    }

    private static String resolveText(Map<String, Object> values, String key, String currentValue) {
        if (!values.containsKey(key)) {
            return currentValue;
        }
        Object raw = values.get(key);
        String submitted = raw == null ? null : raw.toString();
        return (submitted == null || submitted.isEmpty()) ? null : submitted;
    }

    private static Float resolveFloat(Map<String, Object> values, String key, Float currentValue,
            Map<String, String> errors) {
        if (!values.containsKey(key)) {
            return currentValue;
        }
        Object raw = values.get(key);
        String submitted = raw == null ? null : raw.toString();
        if (submitted == null || submitted.isEmpty()) {
            return null;
        }
        try {
            return Float.parseFloat(submitted);
        } catch (NumberFormatException e) {
            errors.put(key, "Must be a number.");
            return currentValue;
        }
    }

    private static Integer resolveInt(Map<String, Object> values, String key, Integer currentValue,
            Map<String, String> errors) {
        if (!values.containsKey(key)) {
            return currentValue;
        }
        Object raw = values.get(key);
        String submitted = raw == null ? null : raw.toString();
        if (submitted == null || submitted.isEmpty()) {
            return null;
        }
        try {
            return Integer.parseInt(submitted);
        } catch (NumberFormatException e) {
            errors.put(key, "Must be a whole number.");
            return currentValue;
        }
    }

    private static Integer resolveFromSet(Map<String, Object> values, String key, Integer currentValue,
            java.util.Set<Integer> allowed, Map<String, String> errors) {
        Integer resolved = resolveInt(values, key, currentValue, errors);
        if (resolved != null && !allowed.contains(resolved)) {
            errors.put(key, "Invalid value.");
            return currentValue;
        }
        return resolved;
    }

    private interface IdExists {
        boolean test(int id);
    }

    private static Integer resolveLookupId(Map<String, Object> values, String key, Integer currentValue,
            IdExists exists, Map<String, String> errors) {
        Integer resolved = resolveInt(values, key, currentValue, errors);
        if (resolved != null && !exists.test(resolved)) {
            errors.put(key, "Unknown value.");
            return currentValue;
        }
        return resolved;
    }

    public PtEvaluationTab createRecord(int patientId, int visitId) {
        if (visits.findPtGeneralMdEvaluationByVisitIdAndPatientId(visitId, patientId).isPresent()) {
            throw new ResponseStatusException(HttpStatus.CONFLICT, "General data already exists for this visit.");
        }
        repository.insertBlankDataPtMovementDisorder(visitId);
        return getGeneral(patientId, visitId, "general-md");
    }
}














package org.nemours.gaitlab.service;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;

import org.nemours.gaitlab.repository.PtEvaluationRepository;
import org.nemours.gaitlab.service.PtEvaluationForm.Field;
import org.nemours.gaitlab.service.PtEvaluationForm.FormRow;
import org.nemours.gaitlab.service.PtEvaluationForm.FormSection;
import org.nemours.gaitlab.service.PtEvaluationForm.Option;
import org.nemours.gaitlab.service.PtEvaluationTab.Row;
import org.nemours.gaitlab.service.PtEvaluationTab.Section;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.server.ResponseStatusException;

/** Ports pt-eval-hat.jsp/modify_pt_hat.jsp - the HAT tab, movement-disorder visits only. Scored
 * once per limb (PtEvaluationRepository.ULLR_OPTIONS: UER, LER, UEL, LEL - that exact order, matching both
 * legacy pages' own column order), via hat_scores.ullr - aligned with production's schema in a
 * Phase 4 audit fix; see PtEvaluationRepository's HAT section javadoc. */
@Service
public class HatService {
    private final PtEvaluationRepository repository;
    private final PtEvaluationRevision revision;

    public HatService(PtEvaluationRepository repository, PtEvaluationRevision revision) {
        this.repository = repository;
        this.revision = revision;
    }

    public PtEvaluationTab build(int patientId, int visitId, String tab) {
        List<Map<String, Object>> rows = repository.findHatScoresByVisitId(visitId);
        if (rows.isEmpty()) {
            return PtEvaluationTab.empty(patientId, visitId, tab, "No HAT scores entered.");
        }
        Map<String, Map<String, Object>> byUllr = byUllr(rows);

        List<String> columns = new ArrayList<>();
        columns.add("");
        PtEvaluationRepository.ULLR_OPTIONS.forEach(ullr -> columns.add(ullr + " Score"));

        List<Row> tabRows = new ArrayList<>();
        for (Map<String, Object> item : repository.findHatItems()) {
            int itemValue = (int) item.get("value");
            List<String> cells = new ArrayList<>();
            for (String ullr : PtEvaluationRepository.ULLR_OPTIONS) {
                Map<String, Object> scores = byUllr.get(ullr);
                Integer selectionId = scores == null ? null : hatColumn(scores, itemValue);
                // Legacy prints nothing at all when unset (pt-eval-hat.jsp:153) - not "--".
                cells.add(selectionId == null ? null : formatSelection(selectionId));
            }
            tabRows.add(new Row((String) item.get("item"), cells));
        }
        Section section = Section.table(null, null, columns, tabRows, null);
        return PtEvaluationTab.of(patientId, visitId, tab, List.of(section));
    }

    private static Map<String, Map<String, Object>> byUllr(List<Map<String, Object>> rows) {
        Map<String, Map<String, Object>> byUllr = new java.util.LinkedHashMap<>();
        for (Map<String, Object> row : rows) {
            byUllr.put((String) row.get("ullr"), row);
        }
        return byUllr;
    }

    private String formatSelection(int selectionId) {
        return repository.findHatSelectionById(selectionId)
                .map(row -> row.get("value") + "=" + row.get("selection"))
                .orElse(null);
    }

    /** hat_item.value (1-7) maps 1:1 to the hatN column, matching HatScore#getSelection/
     * setSelection in the legacy Java model exactly. */
    private static Integer hatColumn(Map<String, Object> scores, int itemValue) {
        return (Integer) scores.get("hat" + itemValue);
    }

    public PtEvaluationForm buildForm(int visitId) {
        List<Map<String, Object>> rows = repository.findHatScoresByVisitId(visitId);
        if (rows.isEmpty()) {
            throw new ResponseStatusException(HttpStatus.NOT_FOUND, "No HAT record exists yet for this visit.");
        }
        Map<String, Map<String, Object>> byUllr = byUllr(rows);

        List<Map<String, Object>> items = repository.findHatItems();
        List<FormRow> formRows = new ArrayList<>();
        for (Map<String, Object> item : items) {
            int itemId = (int) item.get("id");
            int itemValue = (int) item.get("value");
            List<Option> options = toOptions(repository.findHatSelectionsForItem(itemId));
            List<Field> fields = new ArrayList<>();
            for (String ullr : PtEvaluationRepository.ULLR_OPTIONS) {
                Map<String, Object> scores = byUllr.get(ullr);
                Integer current = scores == null ? null : hatColumn(scores, itemValue);
                fields.add(Field.radio(fieldName(ullr, itemId), item.get("item") + " " + ullr,
                        current == null ? null : String.valueOf(current),
                        optionsIncluding(options, current)));
            }
            formRows.add(new FormRow((String) item.get("item"), fields));
        }

        FormSection section = FormSection.grid(null, PtEvaluationRepository.ULLR_OPTIONS, formRows);
        return new PtEvaluationForm("hat", "Edit HAT Values", revision.of(rows), List.of(section));
    }

    /** modify_pt_hat.jsp's own radio field naming: "hi-{ullr}-{itemId}". */
    private static String fieldName(String ullr, int itemId) {
        return "hi-" + ullr + "-" + itemId;
    }

    private static List<Option> toOptions(List<Map<String, Object>> selections) {
        return selections.stream()
                .map(selection -> new Option(String.valueOf(selection.get("id")),
                        selection.get("value") + "=" + selection.get("selection")))
                .toList();
    }

    /** Always includes the row's existing selection as an option even if it's since become
     * inactive - the same current-value-fallback fix applied to Motor Control/Tone/Orthotics
     * earlier in this port. */
    private List<Option> optionsIncluding(List<Option> selections, Integer currentId) {
        List<Option> options = new ArrayList<>(selections);
        boolean hasCurrent = currentId == null
                || options.stream().anyMatch(option -> option.value().equals(String.valueOf(currentId)));
        if (!hasCurrent) {
            repository.findHatSelectionById(currentId).ifPresent(selection -> options.add(0,
                    new Option(String.valueOf(currentId), selection.get("value") + "=" + selection.get("selection"))));
        }
        return options;
    }

    @Transactional
    public PtEvaluationTab save(int patientId, int visitId, String submittedRevision, Map<String, Object> values) {
        List<Map<String, Object>> rows = repository.findHatScoresByVisitId(visitId);
        if (rows.isEmpty()) {
            throw new ResponseStatusException(HttpStatus.NOT_FOUND, "No HAT record exists yet for this visit.");
        }
        if (!revision.of(rows).equals(submittedRevision)) {
            throw new ResponseStatusException(HttpStatus.CONFLICT,
                    "This data has changed since you loaded it. Reload and try again.");
        }
        Map<String, Map<String, Object>> byUllr = byUllr(rows);

        List<Map<String, Object>> items = repository.findHatItems();
        Map<String, String> errors = new java.util.LinkedHashMap<>();
        for (String ullr : PtEvaluationRepository.ULLR_OPTIONS) {
            Map<String, Object> scores = byUllr.get(ullr);
            if (scores == null) {
                continue;
            }
            Integer[] resolved = new Integer[7];
            for (Map<String, Object> item : items) {
                int itemId = (int) item.get("id");
                int itemValue = (int) item.get("value");
                String key = fieldName(ullr, itemId);
                Integer current = hatColumn(scores, itemValue);
                resolved[itemValue - 1] = resolveLookupId(values, key, current,
                        id -> repository.existsHatSelectionForItem(itemId, id), errors);
            }
            if (errors.isEmpty()) {
                repository.updateHatScores((int) scores.get("id"), resolved[0], resolved[1], resolved[2],
                        resolved[3], resolved[4], resolved[5], resolved[6]);
            }
        }

        if (!errors.isEmpty()) {
            throw new PtEvaluationValidationException("Some HAT values are invalid.", errors);
        }

        return build(patientId, visitId, "hat");
    }

    private interface IdExists {
        boolean test(int id);
    }

    private static Integer resolveLookupId(Map<String, Object> values, String key, Integer currentValue,
            IdExists exists, Map<String, String> errors) {
        if (!values.containsKey(key)) {
            return currentValue;
        }
        Object raw = values.get(key);
        String submitted = raw == null ? null : raw.toString();
        if (submitted == null || submitted.isEmpty()) {
            return null;
        }
        Integer resolved;
        try {
            resolved = Integer.parseInt(submitted);
        } catch (NumberFormatException e) {
            errors.put(key, "Must be a whole number.");
            return currentValue;
        }
        if (!exists.test(resolved)) {
            errors.put(key, "Unknown value.");
            return currentValue;
        }
        return resolved;
    }

    public PtEvaluationTab createRecord(int patientId, int visitId) {
        if (!repository.findHatScoresByVisitId(visitId).isEmpty()) {
            throw new ResponseStatusException(HttpStatus.CONFLICT, "HAT data already exists for this visit.");
        }
        repository.insertBlankHatScoresForAllLimbs(visitId);
        return build(patientId, visitId, "hat");
    }
}















package org.nemours.gaitlab.service;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collections;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.function.Function;

import org.nemours.gaitlab.repository.PtEvaluationRepository;
import org.nemours.gaitlab.service.PtEvaluationForm.Field;
import org.nemours.gaitlab.service.PtEvaluationForm.FormRow;
import org.nemours.gaitlab.service.PtEvaluationForm.FormSection;
import org.nemours.gaitlab.service.PtEvaluationForm.Option;
import org.nemours.gaitlab.service.PtEvaluationTab.Row;
import org.nemours.gaitlab.service.PtEvaluationTab.Section;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.web.server.ResponseStatusException;

/** Ports pt-eval-mc.jsp/modify_pt_mc.jsp and pt-eval-tone.jsp/modify_pt_tone.jsp, which are
 * structurally identical pages over the same data_tone table (filtered by mc_tone='mc'/'tone'
 * and joined to a different lookup table for row order). Gait visits only.
 *
 * <p>The two tabs differ in where the R/L Interp dropdown's options come from: Motor Control
 * always reads mc_list; Tone reads the shared "interp" table for ordinary tests but tone_list
 * for "*"-prefixed (interpretation-only) ones - ported from modify_pt_tone.jsp's branch, which
 * modify_pt_mc.jsp doesn't have. All three of mc_list/tone_list/interp are empty in the current
 * seed data, so every Interp dropdown has zero options from the lookup table today (plus
 * whatever the row's own current value contributes) - flagged separately, not bent here. */
@Service
public class MotorControlToneService {
    static final String MOTOR_CONTROL_EMPTY_MESSAGE = "No motor control data has been entered yet for this visit.";
    // Legacy literally has a double space here ("tone  data") - preserved exactly.
    static final String TONE_EMPTY_MESSAGE = "No tone  data has been entered yet for this visit.";
    private static final String FOOTNOTE = "* Interpretation only";
    private static final List<String> COLUMNS = List.of("", "Right", "R Interp", "Left", "L Interp");
    private static final List<String> FORM_COLUMNS = List.of("Right", "R Interp", "Left", "L Interp");

    private final PtEvaluationRepository repository;
    private final PtEvaluationRevision revision;

    public MotorControlToneService(PtEvaluationRepository repository, PtEvaluationRevision revision) {
        this.repository = repository;
        this.revision = revision;
    }

    public PtEvaluationTab buildMotorControl(int patientId, int visitId, String tab) {
        return build(patientId, visitId, tab, repository.findMotorControlByVisitId(visitId), MOTOR_CONTROL_EMPTY_MESSAGE);
    }

    public PtEvaluationTab buildTone(int patientId, int visitId, String tab) {
        return buildTone(patientId, visitId, tab, false);
    }

    /** pt-movement-disorder-eval-tone.jsp's test list is data_tone (filtered to tone_tests.id
     * <= 16 - see findToneByVisitIdForMovementDisorder) PLUS the 6 Upper Extremity rows (pt_ue_
     * tone) appended to the SAME table, right after the tone rows - one shared section, one
     * shared "* Interpretation only" footnote, no "Upper Extremity" sub-title. Legacy's own
     * emptiness check is tones.size()==0 && ueTone==null: only "No tone data entered" when BOTH
     * are missing. */
    public PtEvaluationTab buildTone(int patientId, int visitId, String tab, boolean isMovementDisorder) {
        if (!isMovementDisorder) {
            return build(patientId, visitId, tab, repository.findToneByVisitId(visitId), TONE_EMPTY_MESSAGE);
        }

        List<Map<String, Object>> toneRows = repository.findToneByVisitIdForMovementDisorder(visitId);
        Optional<Map<String, Object>> ueTone = repository.findUeToneByVisitId(visitId);
        if (toneRows.isEmpty() && ueTone.isEmpty()) {
            return PtEvaluationTab.empty(patientId, visitId, tab, TONE_EMPTY_MESSAGE);
        }
        List<Row> rows = new ArrayList<>(toneRows.stream().map(this::toRow).toList());
        ueTone.ifPresent(tone -> rows.addAll(ueToneRows(tone)));
        Section section = Section.table(null, null, COLUMNS, rows, FOOTNOTE);
        return PtEvaluationTab.of(patientId, visitId, tab, List.of(section));
    }

    private static final List<UeToneMuscle> UE_TONE_MUSCLES = List.of(
            new UeToneMuscle("Shoulder extensors", "shoulder_extensors"),
            new UeToneMuscle("Shoulder flexors", "shoulder_flexors"),
            new UeToneMuscle("Elbow extensors", "elbow_extensors"),
            new UeToneMuscle("Elbow flexors", "elbow_flexors"),
            new UeToneMuscle("Wrist extensors", "wrist_extensors"),
            new UeToneMuscle("Wrist flexors", "wrist_flexors"));

    private record UeToneMuscle(String label, String prefix) {
    }

    private List<Row> ueToneRows(Map<String, Object> ueTone) {
        return UE_TONE_MUSCLES.stream()
                .map(muscle -> new Row(muscle.label(), Arrays.asList(
                        str(ueTone.get(muscle.prefix() + "_r")), str(ueTone.get(muscle.prefix() + "_r_interp")),
                        str(ueTone.get(muscle.prefix() + "_l")), str(ueTone.get(muscle.prefix() + "_l_interp")))))
                .toList();
    }

    private PtEvaluationTab build(int patientId, int visitId, String tab, List<Map<String, Object>> data,
            String emptyMessage) {
        if (data.isEmpty()) {
            return PtEvaluationTab.empty(patientId, visitId, tab, emptyMessage);
        }
        List<Row> rows = data.stream().map(this::toRow).toList();
        Section section = Section.table(null, null, COLUMNS, rows, FOOTNOTE);
        return PtEvaluationTab.of(patientId, visitId, tab, List.of(section));
    }

    // Right (0) and Left (2) are the two cell indexes legacy shades grey for a "*"-prefixed
    // (interpretation-only) test; the Interp columns (1, 3) stay visible.
    private static final List<Integer> INTERPRETATION_ONLY_SHADED_CELLS = List.of(0, 2);

    private Row toRow(Map<String, Object> data) {
        String test = (String) data.get("test");
        boolean interpretationOnly = test != null && test.startsWith("*");
        String scoreR = interpretationOnly ? null : PtEvaluationFormat.plainCell((String) data.get("score_r"));
        String scoreL = interpretationOnly ? null : PtEvaluationFormat.plainCell((String) data.get("score_l"));
        String interpR = PtEvaluationFormat.plainCell((String) data.get("interp_r"));
        String interpL = PtEvaluationFormat.plainCell((String) data.get("interp_l"));
        List<Integer> shadedCells = interpretationOnly ? INTERPRETATION_ONLY_SHADED_CELLS : List.of();
        return new Row(test, Arrays.asList(scoreR, interpR, scoreL, interpL), shadedCells, null, null);
    }

    // Legacy literally has a double space here ("Modify  Motor Control") - preserved exactly,
    // same as TONE_EMPTY_MESSAGE above.
    public PtEvaluationForm buildMotorControlForm(int visitId) {
        return buildForm("motor-control", "Modify  Motor Control", repository.findMotorControlByVisitId(visitId),
                test -> repository.findMcListOptions());
    }

    public PtEvaluationForm buildToneForm(int visitId) {
        return buildToneForm(visitId, false);
    }

    public PtEvaluationForm buildToneForm(int visitId, boolean isMovementDisorder) {
        Function<String, List<String>> interpOptionsForTest =
                test -> test.startsWith("*") ? repository.findToneListOptions() : repository.findInterpOptions();
        if (!isMovementDisorder) {
            // Legacy literally has a double space here ("Modify  Tone Data") - preserved
            // exactly. The movement-disorder branch below keeps its own generic "Edit Tone"
            // title - pt-movement-disorder-eval-tone.jsp's own heading is a copy-pasted
            // "PT Evaluation: Tone", not worth porting literally.
            return buildForm("tone", "Modify  Tone Data", repository.findToneByVisitId(visitId),
                    interpOptionsForTest);
        }

        List<Map<String, Object>> toneRows = repository.findToneByVisitIdForMovementDisorder(visitId);
        List<FormRow> rows = new ArrayList<>(toneRows.stream().map(row -> toFormRow(row, interpOptionsForTest))
                .toList());
        Map<String, Object> ueTone = repository.findUeToneByVisitId(visitId)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND,
                        "No tone record exists yet for this visit."));
        rows.addAll(UE_TONE_MUSCLES.stream().map(muscle -> ueToneFormRow(ueTone, muscle)).toList());

        Map<String, Object> revisionSnapshot = new java.util.LinkedHashMap<>();
        revisionSnapshot.put("tone", toneRows);
        revisionSnapshot.put("ueTone", ueTone);
        FormSection section = FormSection.grid(null, FORM_COLUMNS, rows);
        return new PtEvaluationForm("tone", "Edit Tone", revision.of(revisionSnapshot), List.of(section));
    }

    /** Score is a plain text input (modify_pt_movement_disorder_tone.jsp's srX/slX fields);
     * Interp is a dropdown sourced from the same "interp" lookup table the regular Tone rows use
     * for non-"*" tests (its own interpSelect() helper, same options). */
    private FormRow ueToneFormRow(Map<String, Object> ueTone, UeToneMuscle muscle) {
        List<String> interpOptions = repository.findInterpOptions();
        String rInterp = str(ueTone.get(muscle.prefix() + "_r_interp"));
        String lInterp = str(ueTone.get(muscle.prefix() + "_l_interp"));
        Field right = Field.text(muscle.prefix() + "_r", muscle.label() + " Right", str(ueTone.get(muscle.prefix() + "_r")));
        Field rightInterp = Field.select(muscle.prefix() + "_r_interp", muscle.label() + " R Interp", rInterp,
                optionsIncluding(interpOptions, rInterp));
        Field left = Field.text(muscle.prefix() + "_l", muscle.label() + " Left", str(ueTone.get(muscle.prefix() + "_l")));
        Field leftInterp = Field.select(muscle.prefix() + "_l_interp", muscle.label() + " L Interp", lInterp,
                optionsIncluding(interpOptions, lInterp));
        return new FormRow(muscle.label(), Arrays.asList(right, rightInterp, left, leftInterp));
    }

    private PtEvaluationForm buildForm(String tab, String title, List<Map<String, Object>> data,
            Function<String, List<String>> interpOptionsForTest) {
        List<FormRow> rows = data.stream().map(row -> toFormRow(row, interpOptionsForTest)).toList();
        FormSection section = FormSection.grid(null, FORM_COLUMNS, rows);
        return new PtEvaluationForm(tab, title, revision.of(data), List.of(section));
    }

    private FormRow toFormRow(Map<String, Object> data, Function<String, List<String>> interpOptionsForTest) {
        String test = (String) data.get("test");
        int id = (Integer) data.get("id");
        boolean interpretationOnly = test != null && test.startsWith("*");
        String interpR = str(data.get("interp_r"));
        String interpL = str(data.get("interp_l"));
        List<String> interpOptions = interpOptionsForTest.apply(test);

        Field scoreR = interpretationOnly ? null
                : Field.text("score_r_" + id, test + " Right", str(data.get("score_r")));
        Field interpRField = Field.select("interp_r_" + id, test + " R Interp", interpR,
                optionsIncluding(interpOptions, interpR));
        Field scoreL = interpretationOnly ? null
                : Field.text("score_l_" + id, test + " Left", str(data.get("score_l")));
        Field interpLField = Field.select("interp_l_" + id, test + " L Interp", interpL,
                optionsIncluding(interpOptions, interpL));

        return new FormRow(test, Arrays.asList(scoreR, interpRField, scoreL, interpLField));
    }

    /** A leading blank option (so the field can be cleared), unless the lookup table already has
     * its own - mc_list/tone_list/interp all have one - plus the row's current value if it isn't
     * already in that list, since existing data like "mild" or "2+" predates these lookups. */
    private static List<Option> optionsIncluding(List<String> lookupValues, String currentValue) {
        List<Option> options = new ArrayList<>();
        if (!lookupValues.contains("")) {
            options.add(new Option("", ""));
        }
        for (String value : lookupValues) {
            options.add(new Option(value, value));
        }
        if (currentValue != null && !currentValue.isEmpty() && !lookupValues.contains(currentValue)) {
            options.add(new Option(currentValue, currentValue));
        }
        return Collections.unmodifiableList(options);
    }

    private static String str(Object value) {
        return value == null ? null : value.toString();
    }

    public PtEvaluationTab saveMotorControl(int patientId, int visitId, String submittedRevision,
            Map<String, Object> values) {
        return save(patientId, visitId, "motor-control", () -> buildMotorControl(patientId, visitId, "motor-control"),
                repository.findMotorControlByVisitId(visitId), submittedRevision, values);
    }

    public PtEvaluationTab saveTone(int patientId, int visitId, String submittedRevision,
            Map<String, Object> values) {
        return saveTone(patientId, visitId, submittedRevision, values, false);
    }

    public PtEvaluationTab saveTone(int patientId, int visitId, String submittedRevision,
            Map<String, Object> values, boolean isMovementDisorder) {
        if (!isMovementDisorder) {
            return save(patientId, visitId, "tone", () -> buildTone(patientId, visitId, "tone"),
                    repository.findToneByVisitId(visitId), submittedRevision, values);
        }

        List<Map<String, Object>> toneRows = repository.findToneByVisitIdForMovementDisorder(visitId);
        Map<String, Object> ueTone = repository.findUeToneByVisitId(visitId)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND,
                        "No tone record exists yet for this visit."));
        Map<String, Object> revisionSnapshot = new java.util.LinkedHashMap<>();
        revisionSnapshot.put("tone", toneRows);
        revisionSnapshot.put("ueTone", ueTone);
        if (!revision.of(revisionSnapshot).equals(submittedRevision)) {
            throw new ResponseStatusException(HttpStatus.CONFLICT,
                    "This data has changed since you loaded it. Reload and try again.");
        }

        for (Map<String, Object> row : toneRows) {
            int id = (Integer) row.get("id");
            String test = (String) row.get("test");
            boolean interpretationOnly = test != null && test.startsWith("*");
            String interpR = resolve(values, "interp_r_" + id, str(row.get("interp_r")));
            String interpL = resolve(values, "interp_l_" + id, str(row.get("interp_l")));
            if (interpretationOnly) {
                repository.updateInterpOnly(id, interpR, interpL);
            } else {
                String scoreR = resolve(values, "score_r_" + id, str(row.get("score_r")));
                String scoreL = resolve(values, "score_l_" + id, str(row.get("score_l")));
                repository.updateScoreAndInterp(id, scoreR, interpR, scoreL, interpL);
            }
        }

        Map<String, String> ueUpdates = new java.util.LinkedHashMap<>();
        for (UeToneMuscle muscle : UE_TONE_MUSCLES) {
            for (String field : List.of("_r", "_r_interp", "_l", "_l_interp")) {
                String key = muscle.prefix() + field;
                ueUpdates.put(key, resolve(values, key, str(ueTone.get(key))));
            }
        }
        repository.updateUeTone((int) ueTone.get("id"), ueUpdates);

        return buildTone(patientId, visitId, "tone", true);
    }

    /** No revision check exists in legacy at all (a raw unconditional UPDATE) - this 409 is a
     * deliberate addition, not a ported behavior. Interpretation-only rows only ever touch
     * interp_r/interp_l: there's no score input to submit for them in the first place (legacy
     * replaces that cell with a grey filler), so an absent score field for those rows is
     * expected, not a client error. */
    private PtEvaluationTab save(int patientId, int visitId, String tab,
            java.util.function.Supplier<PtEvaluationTab> rebuild, List<Map<String, Object>> currentData,
            String submittedRevision, Map<String, Object> values) {
        if (!revision.of(currentData).equals(submittedRevision)) {
            throw new ResponseStatusException(HttpStatus.CONFLICT,
                    "This data has changed since you loaded it. Reload and try again.");
        }
        for (Map<String, Object> row : currentData) {
            int id = (Integer) row.get("id");
            String test = (String) row.get("test");
            boolean interpretationOnly = test != null && test.startsWith("*");
            String interpR = resolve(values, "interp_r_" + id, str(row.get("interp_r")));
            String interpL = resolve(values, "interp_l_" + id, str(row.get("interp_l")));
            if (interpretationOnly) {
                repository.updateInterpOnly(id, interpR, interpL);
            } else {
                String scoreR = resolve(values, "score_r_" + id, str(row.get("score_r")));
                String scoreL = resolve(values, "score_l_" + id, str(row.get("score_l")));
                repository.updateScoreAndInterp(id, scoreR, interpR, scoreL, interpL);
            }
        }
        return rebuild.get();
    }

    /** A missing key leaves the field untouched (so a client that only sends changed fields
     * still works); a present key with "" clears it; anything else is the new value. */
    private static String resolve(Map<String, Object> values, String key, String currentValue) {
        if (!values.containsKey(key)) {
            return currentValue;
        }
        String submitted = str(values.get(key));
        return (submitted == null || submitted.isEmpty()) ? null : submitted;
    }

    public PtEvaluationTab createMotorControlRecord(int patientId, int visitId) {
        if (!repository.findMotorControlByVisitId(visitId).isEmpty()) {
            throw new ResponseStatusException(HttpStatus.CONFLICT, "Motor Control data already exists for this visit.");
        }
        repository.findMcTestNames().forEach(test -> repository.insertBlankRow(visitId, test, "mc"));
        return buildMotorControl(patientId, visitId, "motor-control");
    }

    public PtEvaluationTab createToneRecord(int patientId, int visitId) {
        return createToneRecord(patientId, visitId, false);
    }

    public PtEvaluationTab createToneRecord(int patientId, int visitId, boolean isMovementDisorder) {
        boolean hasTone = !repository.findToneByVisitId(visitId).isEmpty();
        boolean hasUeTone = isMovementDisorder && repository.findUeToneByVisitId(visitId).isPresent();
        if (hasTone || hasUeTone) {
            throw new ResponseStatusException(HttpStatus.CONFLICT, "Tone data already exists for this visit.");
        }
        repository.findToneTestNames().forEach(test -> repository.insertBlankRow(visitId, test, "tone"));
        if (isMovementDisorder) {
            repository.insertBlankUeTone(visitId);
        }
        return buildTone(patientId, visitId, "tone", isMovementDisorder);
    }
}
















package org.nemours.gaitlab.service;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.Map;
import java.util.Optional;

import org.nemours.gaitlab.repository.PtEvaluationRepository;
import org.nemours.gaitlab.repository.VisitRepository;
import org.nemours.gaitlab.repository.VisitRepository.PtGeneralMdEvaluationRow;
import org.nemours.gaitlab.service.PtEvaluationForm.Field;
import org.nemours.gaitlab.service.PtEvaluationForm.FormSection;
import org.nemours.gaitlab.service.PtEvaluationForm.Option;
import org.nemours.gaitlab.service.PtEvaluationTab.Row;
import org.nemours.gaitlab.service.PtEvaluationTab.RowActions;
import org.nemours.gaitlab.service.PtEvaluationTab.Section;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.server.ResponseStatusException;

/** Ports lab/data/pt-eval-orth.jsp's orthoses list and insert_orth.jsp/modify_orth.jsp/
 * delete_orth.jsp. Unlike the other tabs, the legacy page shows no text at all when there is no
 * data - just an empty page - so emptyMessage is null here rather than a copied legacy string.
 * Each row's edit/delete icons are admin-only in legacy (same flag, not independently gated), so
 * both RowActions booleans mirror isAdmin - enforced again at the write endpoints themselves,
 * since legacy's own insert/modify/delete pages have no check beyond being on the
 * admin-restricted page in the first place.
 *
 * <p>Movement-disorder visits add a "Comments" block below the (shared) orthoses list -
 * data_pt_movement_disorder.orthoses, a free-text textarea with no admin gate on viewing/editing
 * it, only on creating the record in the first place (pt-eval-orth.jsp:206-228). It's the exact
 * same data_pt_movement_disorder row general-md/record creates, so this reuses
 * VisitRepository/PtEvaluationRepository directly rather than re-deriving that existence check. */
@Service
public class OrthosesService {
    private static final List<Option> SIDE_OPTIONS = List.of(
            new Option("Left", "Left"), new Option("Right", "Right"), new Option("Both", "Both"));
    private static final List<String> VALID_SIDES = List.of("", "Left", "Right", "Both");

    private final PtEvaluationRepository repository;
    private final VisitRepository visits;
    private final PtEvaluationRevision revision;

    public OrthosesService(PtEvaluationRepository repository, VisitRepository visits,
            PtEvaluationRevision revision) {
        this.repository = repository;
        this.visits = visits;
        this.revision = revision;
    }

    public PtEvaluationTab build(int patientId, int visitId, String tab, boolean isAdmin) {
        return build(patientId, visitId, tab, isAdmin, false);
    }

    public PtEvaluationTab build(int patientId, int visitId, String tab, boolean isAdmin,
            boolean isMovementDisorder) {
        List<Map<String, Object>> orthoses = repository.findOrthosesByVisitId(visitId);
        if (!isMovementDisorder) {
            if (orthoses.isEmpty()) {
                return PtEvaluationTab.empty(patientId, visitId, tab, null);
            }
            return PtEvaluationTab.of(patientId, visitId, tab, List.of(orthosesSection(orthoses, isAdmin)));
        }

        Optional<PtGeneralMdEvaluationRow> mdRow = findMdRow(patientId, visitId);
        if (orthoses.isEmpty() && mdRow.isEmpty()) {
            return PtEvaluationTab.empty(patientId, visitId, tab, null);
        }
        List<Section> sections = new ArrayList<>();
        if (!orthoses.isEmpty()) {
            sections.add(orthosesSection(orthoses, isAdmin));
        }
        sections.add(Section.text("Comments", mdRow.map(PtGeneralMdEvaluationRow::getOrthoses).orElse(null)));
        return PtEvaluationTab.of(patientId, visitId, tab, sections);
    }

    private Optional<PtGeneralMdEvaluationRow> findMdRow(int patientId, int visitId) {
        return visits.findPtGeneralMdEvaluationByVisitIdAndPatientId(visitId, patientId);
    }

    private Section orthosesSection(List<Map<String, Object>> orthoses, boolean isAdmin) {
        RowActions rowActions = new RowActions(isAdmin, isAdmin);
        List<Row> rows = orthoses.stream()
                .map(o -> new Row((String) o.get("orthosis"), Arrays.asList((String) o.get("side")),
                        List.of(), (Integer) o.get("id"), rowActions))
                .toList();
        return Section.table(null, null, List.of("Orthoses", "Side"), rows, null);
    }

    /** true only for movement-disorder visits where the data_pt_movement_disorder row already
     * exists - the Comments textarea/Save is visible and editable by anyone once that's true, no
     * admin gate (pt-eval-orth.jsp:217-227). */
    public boolean commentsEditable(int patientId, int visitId) {
        return findMdRow(patientId, visitId).isPresent();
    }

    /** admin-only, and only when no data_pt_movement_disorder row exists yet - matches the
     * Create New Record button's own gate on that same block. */
    public boolean commentsCreatable(int patientId, int visitId, boolean isAdmin) {
        return isAdmin && findMdRow(patientId, visitId).isEmpty();
    }

    public PtEvaluationForm buildCommentsForm(int patientId, int visitId) {
        PtGeneralMdEvaluationRow row = findMdRow(patientId, visitId)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND,
                        "No general record exists yet for this visit."));
        Field orthoses = Field.textarea("orthoses", "Comments", row.getOrthoses());
        return new PtEvaluationForm("orthotics", "Comments", revision.of(row.getOrthoses()),
                List.of(FormSection.stack(null, List.of(orthoses))));
    }

    @Transactional
    public PtEvaluationTab saveComments(int patientId, int visitId, String submittedRevision,
            Map<String, Object> values) {
        PtGeneralMdEvaluationRow row = findMdRow(patientId, visitId)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND,
                        "No general record exists yet for this visit."));
        if (!revision.of(row.getOrthoses()).equals(submittedRevision)) {
            throw new ResponseStatusException(HttpStatus.CONFLICT,
                    "This data has changed since you loaded it. Reload and try again.");
        }
        Object raw = values.get("orthoses");
        String orthoses = raw == null ? null : raw.toString();
        repository.updateMovementDisorderOrthosesComment(row.getDataPtMdId(), orthoses == null || orthoses.isEmpty() ? null
                : orthoses);
        return build(patientId, visitId, "orthotics", true, true);
    }

    /** Creates the same data_pt_movement_disorder row general-md/record does - they're the same
     * row, just reached from a different tab. */
    public PtEvaluationTab createCommentsRecord(int patientId, int visitId) {
        if (findMdRow(patientId, visitId).isPresent()) {
            throw new ResponseStatusException(HttpStatus.CONFLICT, "General data already exists for this visit.");
        }
        repository.insertBlankDataPtMovementDisorder(visitId);
        return build(patientId, visitId, "orthotics", true, true);
    }

    /** No prior state exists for a brand-new orthosis, so the revision is just a fixed marker
     * for this visit rather than a hash of real data - it still round-trips through the same
     * {revision, values} contract as every other save, it just never legitimately conflicts. */
    public PtEvaluationForm buildNewForm(int visitId) {
        Field orthosis = requiredOrthosisField(null, repository.findOrthosisOptions());
        Field side = new Field("side", "Side", "radio", null, SIDE_OPTIONS, false, null, null, null, null, null);
        FormSection section = FormSection.stack(null, List.of(orthosis, side));
        return new PtEvaluationForm("orthotics", "Add Orthosis to Visit", revision.of("new:" + visitId),
                List.of(section));
    }

    public PtEvaluationForm buildEditForm(int visitId, int id) {
        Map<String, Object> row = repository.findByIdAndVisitId(id, visitId)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));
        Field orthosis = requiredOrthosisField((String) row.get("orthosis"), repository.findOrthosisOptions());
        Field side = new Field("side", "Side", "radio", row.get("side"), SIDE_OPTIONS, false, null, null, null,
                null, null);
        FormSection section = FormSection.stack(null, List.of(orthosis, side));
        return new PtEvaluationForm("orthotics", "Modify Patient Orthosis", revision.of(row), List.of(section));
    }

    /** required: true, since the server already 400s on a blank orthosis - legacy's select has
     * no blank option either (insert_orth.jsp/modify_orth.jsp). Includes the current value even
     * when it isn't in the lookup, the same fallback used for Motor Control/Tone's Interp
     * dropdowns, so an edit never shows a blank select for a legitimately stored value. */
    private static Field requiredOrthosisField(String currentValue, List<String> lookupValues) {
        List<Option> options = new ArrayList<>();
        for (String value : lookupValues) {
            options.add(new Option(value, value));
        }
        if (currentValue != null && !currentValue.isEmpty() && !lookupValues.contains(currentValue)) {
            options.add(new Option(currentValue, currentValue));
        }
        return new Field("orthosis", "Orthosis", "select", currentValue, options, true, null, null, null, null,
                null);
    }

    public PtEvaluationTab createOrthosis(int patientId, int visitId, boolean isAdmin, String submittedRevision,
            Map<String, Object> values) {
        if (!revision.of("new:" + visitId).equals(submittedRevision)) {
            throw new ResponseStatusException(HttpStatus.CONFLICT,
                    "This form is stale. Reload and try again.");
        }
        String orthosis = validated(values, null);
        String side = validatedSide(values);
        repository.insert(visitId, orthosis, side);
        return build(patientId, visitId, "orthotics", isAdmin);
    }

    public PtEvaluationTab updateOrthosis(int patientId, int visitId, int id, boolean isAdmin,
            String submittedRevision, Map<String, Object> values) {
        Map<String, Object> row = repository.findByIdAndVisitId(id, visitId)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));
        if (!revision.of(row).equals(submittedRevision)) {
            throw new ResponseStatusException(HttpStatus.CONFLICT,
                    "This data has changed since you loaded it. Reload and try again.");
        }
        String orthosis = validated(values, (String) row.get("orthosis"));
        String side = validatedSide(values);
        repository.update(id, visitId, orthosis, side);
        return build(patientId, visitId, "orthotics", isAdmin);
    }

    /** Must be a real orthoses.orthosis value - legacy's select is select-only, no free text
     * (insert_orth.jsp:221, modify_orth.jsp:224). On an edit, the row's own pre-existing value is
     * also accepted even if it predates the lookup table, so an unrelated save doesn't fail on
     * data this port didn't create. */
    private String validated(Map<String, Object> values, String existingValue) {
        Object raw = values.get("orthosis");
        String orthosis = raw == null ? null : raw.toString();
        if (orthosis == null || orthosis.isBlank()) {
            throw new PtEvaluationValidationException("An orthosis is required.",
                    Map.of("orthosis", "An orthosis is required."));
        }
        if (!orthosis.equals(existingValue) && !repository.findOrthosisOptions().contains(orthosis)) {
            throw new PtEvaluationValidationException("Unknown orthosis.",
                    Map.of("orthosis", "Must be one of the listed orthoses."));
        }
        return orthosis;
    }

    private static String validatedSide(Map<String, Object> values) {
        Object raw = values.get("side");
        String side = raw == null ? "" : raw.toString();
        if (!VALID_SIDES.contains(side)) {
            throw new PtEvaluationValidationException("Invalid side.", Map.of("side", "Must be Left, Right, or Both."));
        }
        return side;
    }

    public PtEvaluationTab deleteOrthosis(int patientId, int visitId, int id, boolean isAdmin) {
        repository.findByIdAndVisitId(id, visitId)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));
        repository.deleteByIdAndVisitId(id, visitId);
        return build(patientId, visitId, "orthotics", isAdmin);
    }
}














package org.nemours.gaitlab.service;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

import org.nemours.gaitlab.repository.PtEvaluationRepository;
import org.nemours.gaitlab.service.PtEvaluationForm.Field;
import org.nemours.gaitlab.service.PtEvaluationForm.FormSection;
import org.nemours.gaitlab.service.PtEvaluationForm.Option;
import org.nemours.gaitlab.service.PtEvaluationTab.Row;
import org.nemours.gaitlab.service.PtEvaluationTab.Section;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.server.ResponseStatusException;

/**
 * Ports lab/data/pt-eval-pain.jsp (view) and modify_pt_pain.jsp (edit), Gait visits only.
 * hasData tracks whether a data_pain row exists at all - once it does, the legacy page renders
 * every field (even if individually blank), so that's the only emptiness check, unlike the
 * GMFM-D-driven tabs.
 *
 * <p>The legacy page is genuinely one single HTML TABLE (lines 144-209) - factors/activities are
 * bullet lists rendered INSIDE that table's cells, not separate sections. Ported as one table
 * section with the list items newline-joined into their row's single cell, matching legacy's
 * exact row order and label text (including the colons on the first three rows only).
 */
@Service
public class PainService {
    static final String EMPTY_MESSAGE = "No pain data entered.";
    private static final List<String> LABEL_VALUE_COLUMNS = List.of("", "");
    private static final List<String> LEVEL_FIELDS = List.of("currentLevel", "lowestLevel", "highestLevel");

    private final PtEvaluationRepository repository;
    private final PtEvaluationRevision revision;

    public PainService(PtEvaluationRepository repository, PtEvaluationRevision revision) {
        this.repository = repository;
        this.revision = revision;
    }

    public PtEvaluationTab build(int patientId, int visitId, String tab) {
        return repository.findPainByVisitId(visitId)
                .map(row -> buildTab(patientId, visitId, tab, row))
                .orElseGet(() -> PtEvaluationTab.empty(patientId, visitId, tab, EMPTY_MESSAGE));
    }

    private PtEvaluationTab buildTab(int patientId, int visitId, String tab, Map<String, Object> row) {
        int dataPainId = (Integer) row.get("id");

        Section section = Section.table(null, null, LABEL_VALUE_COLUMNS, List.of(
                new Row("Pain assessment method:", Arrays.asList(str(row, "assessment_method"))),
                new Row("Aggravating factors:",
                        Arrays.asList(joinOrNull(repository.findFactorLabels(dataPainId)))),
                new Row("Activities impacted:",
                        Arrays.asList(joinOrNull(repository.findActivitiesImpactedLabels(dataPainId)))),
                new Row("Current Level", Arrays.asList(str(row, "current_level"))),
                new Row("Lowest Level", Arrays.asList(str(row, "lowest_level"))),
                new Row("Highest Level", Arrays.asList(str(row, "highest_level"))),
                new Row("Location of pain", Arrays.asList(str(row, "location"))),
                new Row("Frequency of pain", Arrays.asList(str(row, "frequency"))),
                new Row("Duration of pain", Arrays.asList(str(row, "duration")))), null);

        return PtEvaluationTab.of(patientId, visitId, tab, List.of(section));
    }

    /** legacy renders an empty <ul></ul> (no visible content) when there are no items - null
     * cell, not an empty string, matching every other "nothing on file" cell in this port. */
    private static String joinOrNull(List<String> items) {
        return items.isEmpty() ? null : String.join("\n", items);
    }

    private String str(Map<String, Object> row, String key) {
        Object value = row.get(key);
        return value == null ? null : value.toString();
    }

    public PtEvaluationForm buildForm(int patientId, int visitId) {
        Map<String, Object> row = repository.findPainByVisitId(visitId)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND,
                        "No pain record exists yet for this visit."));
        int dataPainId = (Integer) row.get("id");
        return buildForm(row, dataPainId);
    }

    private PtEvaluationForm buildForm(Map<String, Object> row, int dataPainId) {
        List<Integer> factorIds = repository.findFactorIds(dataPainId);
        List<Integer> activityIds = repository.findActivitiesImpactedIds(dataPainId);

        Field assessmentMethod = Field.select("assessmentMethod", "Pain assessment method",
                str(row, "pain_assessment_method_id"), toOptions(repository.findAssessmentMethodOptions()));
        Field factors = new Field("factors", "Aggravating factors", "multiselect",
                factorIds.stream().map(String::valueOf).toList(), toOptions(repository.findFactorOptions()),
                false, null, null, null, null, null);
        Field activitiesImpacted = new Field("activitiesImpacted", "Activities impacted", "multiselect",
                activityIds.stream().map(String::valueOf).toList(),
                toOptions(repository.findActivitiesImpactedOptions()), false, null, null, null, null, null);
        Field currentLevel = levelField("currentLevel", "Current Level", str(row, "current_level"));
        Field lowestLevel = levelField("lowestLevel", "Lowest Level", str(row, "lowest_level"));
        Field highestLevel = levelField("highestLevel", "Highest Level", str(row, "highest_level"));
        Field location = Field.text("location", "Location of pain", str(row, "location"));
        Field frequency = Field.text("frequency", "Frequency of pain", str(row, "frequency"));
        Field duration = Field.text("duration", "Duration of pain", str(row, "duration"));

        FormSection section = FormSection.stack(null, List.of(assessmentMethod, factors, activitiesImpacted,
                currentLevel, lowestLevel, highestLevel, location, frequency, duration));
        return new PtEvaluationForm("pain", "Edit Pain", revision.of(snapshot(row, factorIds, activityIds)),
                List.of(section));
    }

    private static Field levelField(String name, String label, String value) {
        List<Option> options = new ArrayList<>();
        for (int i = 0; i <= 10; i++) {
            options.add(new Option(String.valueOf(i), String.valueOf(i)));
        }
        return new Field(name, label, "radio", value, options, false, 0.0, 10.0, 1.0, null, null);
    }

    private static List<Option> toOptions(List<Map<String, Object>> rows) {
        return rows.stream()
                .map(row -> new Option(String.valueOf(row.get("id")), (String) row.get("label")))
                .toList();
    }

    /** A stable, order-independent view of the editable data for the revision hash, since
     * factor/activity ids come back from two separate queries whose row order isn't otherwise
     * guaranteed to be identical between the GET and the later PUT's freshly-read copy. */
    private static Map<String, Object> snapshot(Map<String, Object> row, List<Integer> factorIds,
            List<Integer> activityIds) {
        Map<String, Object> snapshot = new LinkedHashMap<>(row);
        snapshot.put("factorIds", factorIds.stream().sorted().toList());
        snapshot.put("activityIds", activityIds.stream().sorted().toList());
        return snapshot;
    }

    @Transactional
    public PtEvaluationTab save(int patientId, int visitId, String submittedRevision, Map<String, Object> values) {
        Map<String, Object> row = repository.findPainByVisitId(visitId)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND,
                        "No pain record exists yet for this visit."));
        int dataPainId = (Integer) row.get("id");
        List<Integer> currentFactorIds = repository.findFactorIds(dataPainId);
        List<Integer> currentActivityIds = repository.findActivitiesImpactedIds(dataPainId);

        if (!revision.of(snapshot(row, currentFactorIds, currentActivityIds)).equals(submittedRevision)) {
            throw new ResponseStatusException(HttpStatus.CONFLICT,
                    "This data has changed since you loaded it. Reload and try again.");
        }

        Map<String, String> errors = new LinkedHashMap<>();
        Integer assessmentMethodId = resolveLookupId(values, "assessmentMethod",
                (Integer) row.get("pain_assessment_method_id"), repository::existsAssessmentMethod, errors);
        Integer currentLevel = resolveLevel(values, "currentLevel", (Integer) row.get("current_level"), errors);
        Integer lowestLevel = resolveLevel(values, "lowestLevel", (Integer) row.get("lowest_level"), errors);
        Integer highestLevel = resolveLevel(values, "highestLevel", (Integer) row.get("highest_level"), errors);
        List<Integer> factorIds = resolveIdList(values, "factors", currentFactorIds, repository::existsFactor, errors);
        List<Integer> activityIds = resolveIdList(values, "activitiesImpacted", currentActivityIds,
                repository::existsActivityImpacted, errors);
        String location = resolveText(values, "location", str(row, "location"));
        String frequency = resolveText(values, "frequency", str(row, "frequency"));
        String duration = resolveText(values, "duration", str(row, "duration"));

        if (!errors.isEmpty()) {
            throw new PtEvaluationValidationException("Some pain fields are invalid.", errors);
        }

        repository.updateRecord(dataPainId, assessmentMethodId, currentLevel, lowestLevel, highestLevel, location,
                frequency, duration);
        repository.replaceFactorLinks(dataPainId, factorIds);
        repository.replaceActivitiesImpactedLinks(dataPainId, activityIds);

        return build(patientId, visitId, "pain");
    }

    private static String resolveText(Map<String, Object> values, String key, String currentValue) {
        if (!values.containsKey(key)) {
            return currentValue;
        }
        Object raw = values.get(key);
        String submitted = raw == null ? null : raw.toString();
        return (submitted == null || submitted.isEmpty()) ? null : submitted;
    }

    private static Integer resolveLevel(Map<String, Object> values, String key, Integer currentValue,
            Map<String, String> errors) {
        if (!values.containsKey(key)) {
            return currentValue;
        }
        Object raw = values.get(key);
        String submitted = raw == null ? null : raw.toString();
        if (submitted == null || submitted.isEmpty()) {
            return null;
        }
        try {
            int level = Integer.parseInt(submitted);
            if (level < 0 || level > 10) {
                errors.put(key, "Must be between 0 and 10.");
                return currentValue;
            }
            return level;
        } catch (NumberFormatException e) {
            errors.put(key, "Must be a whole number from 0 to 10.");
            return currentValue;
        }
    }

    private interface IdExists {
        boolean test(int id);
    }

    private static Integer resolveLookupId(Map<String, Object> values, String key, Integer currentValue,
            IdExists exists, Map<String, String> errors) {
        if (!values.containsKey(key)) {
            return currentValue;
        }
        Object raw = values.get(key);
        String submitted = raw == null ? null : raw.toString();
        if (submitted == null || submitted.isEmpty()) {
            return null;
        }
        try {
            int id = Integer.parseInt(submitted);
            if (!exists.test(id)) {
                errors.put(key, "Unknown value.");
                return currentValue;
            }
            return id;
        } catch (NumberFormatException e) {
            errors.put(key, "Invalid value.");
            return currentValue;
        }
    }

    private static List<Integer> resolveIdList(Map<String, Object> values, String key, List<Integer> currentIds,
            IdExists exists, Map<String, String> errors) {
        if (!values.containsKey(key)) {
            return currentIds;
        }
        Object raw = values.get(key);
        if (raw == null) {
            return List.of();
        }
        if (!(raw instanceof List<?> submitted)) {
            errors.put(key, "Must be a list of values.");
            return currentIds;
        }
        List<Integer> result = new ArrayList<>();
        for (Object item : submitted) {
            String s = item == null ? null : item.toString();
            if (s == null || s.isEmpty()) {
                continue;
            }
            try {
                int id = Integer.parseInt(s);
                if (!exists.test(id)) {
                    errors.put(key, "Contains an unknown value.");
                    continue;
                }
                result.add(id);
            } catch (NumberFormatException e) {
                errors.put(key, "Must be a list of numeric ids.");
            }
        }
        return result;
    }

    public PtEvaluationTab createRecord(int patientId, int visitId) {
        if (repository.findPainByVisitId(visitId).isPresent()) {
            throw new ResponseStatusException(HttpStatus.CONFLICT, "Pain data already exists for this visit.");
        }
        repository.insertBlankRecord(visitId);
        return build(patientId, visitId, "pain");
    }
}














package org.nemours.gaitlab.service;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

import org.nemours.gaitlab.repository.PtEvaluationRepository;
import org.nemours.gaitlab.service.PtEvaluationForm.Field;
import org.nemours.gaitlab.service.PtEvaluationForm.FormRow;
import org.nemours.gaitlab.service.PtEvaluationForm.FormSection;
import org.nemours.gaitlab.service.PtEvaluationTab.ColumnGroup;
import org.nemours.gaitlab.service.PtEvaluationTab.Row;
import org.nemours.gaitlab.service.PtEvaluationTab.Section;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.server.ResponseStatusException;

/** Ports lab/data/pt-eval-prom.jsp (PROM/Strength tab), Gait visits only. */
@Service
public class PromStrengthService {
    static final String EMPTY_MESSAGE = "No general data has been entered yet for this visit.";

    private final PtEvaluationRepository repository;
    private final PtEvaluationRevision revision;

    public PromStrengthService(PtEvaluationRepository repository, PtEvaluationRevision revision) {
        this.repository = repository;
        this.revision = revision;
    }

    public PtEvaluationTab build(int patientId, int visitId, String tab) {
        return build(patientId, visitId, tab, false);
    }

    /** pt-sections.jsp swaps the backing table to data_movement_disorder_prom for
     * movement-disorder visits - a genuinely different, smaller field set (modify_pt_prom.jsp's
     * own isMovementDisorder branch), not just a different table name for the same columns. */
    public PtEvaluationTab build(int patientId, int visitId, String tab, boolean isMovementDisorder) {
        if (isMovementDisorder) {
            return repository.findMdByVisitId(visitId)
                    .map(row -> PtEvaluationTab.of(patientId, visitId, tab, List.of(buildMdSection(row))))
                    .orElseGet(() -> PtEvaluationTab.empty(patientId, visitId, tab, EMPTY_MESSAGE));
        }
        return repository.findPromByVisitId(visitId)
                .map(row -> PtEvaluationTab.of(patientId, visitId, tab, List.of(buildSection(row))))
                .orElseGet(() -> PtEvaluationTab.empty(patientId, visitId, tab, EMPTY_MESSAGE));
    }

    private Section buildMdSection(Map<String, Object> row) {
        List<ColumnGroup> columnGroups = List.of(
                new ColumnGroup("", 1), new ColumnGroup("PROM", 2), new ColumnGroup("Strength", 2));
        List<String> columns = List.of("", "Right", "Left", "Right", "Left");

        List<Row> rows = new ArrayList<>();
        rows.add(angleRow(row, "Hip Abd", "hip_abd"));
        rows.add(angleRow(row, "Knee Ext", "knee_ext"));
        rows.add(popAngleRow(row));
        rows.add(angleRow(row, "Dorsi (flex)", "dorsi_f"));
        rows.add(angleOnlyRow(row, "Dorsi (ext)", "dorsi_e"));
        rows.add(angleRow(row, "Shoulder Abduction", "shoulder_abduction"));
        rows.add(angleRow(row, "Elbow Extension", "elbow_extension"));
        rows.add(angleRow(row, "Wrist Extension", "wrist_extension"));

        return Section.table(null, columnGroups, columns, rows, null);
    }

    private Section buildSection(Map<String, Object> row) {
        List<ColumnGroup> columnGroups = List.of(
                new ColumnGroup("", 1), new ColumnGroup("PROM", 2), new ColumnGroup("Strength", 2));
        List<String> columns = List.of("", "Right", "Left", "Right", "Left");

        List<Row> rows = new ArrayList<>();
        rows.add(angleRow(row, "Hip Flex", "hip_flex"));
        rows.add(angleRow(row, "Hip Ext", "hip_ext"));
        rows.add(angleRow(row, "Hip Abd", "hip_abd"));
        rows.add(angleOnlyRow(row, "Hip Int Rot", "hip_int_rot"));
        rows.add(angleOnlyRow(row, "Hip Ext Rot", "hip_ext_rot"));
        rows.add(angleOnlyRow(row, "Ryder Test", "ryder_test"));
        rows.add(angleRow(row, "Knee Ext", "knee_ext"));
        rows.add(angleOnlyRow(row, "Extensor Lag", "extensor_lag"));
        rows.add(angleRow(row, "Knee Flex", "knee_flex"));
        rows.add(popAngleRow(row));
        rows.add(angleOnlyRow(row, "Bilateral pop angle", "bilateral_pop"));
        rows.add(angleOnlyRow(row, "Ely Test", "ely"));
        rows.add(angleRow(row, "Dorsi (flex)", "dorsi_f"));
        rows.add(angleOnlyRow(row, "Dorsi (ext)", "dorsi_e"));
        rows.add(angleRow(row, "Plantar", "plantar"));
        rows.add(angleRow(row, "Ankle Inv", "ankle_inv"));
        rows.add(angleRow(row, "Ankle Ever", "ankle_ever"));
        rows.add(textOnlyRow(row, "TMA", "tma"));
        rows.add(textOnlyRow(row, "TFA", "tfa"));
        rows.add(textOnlyRow(row, "FF AB/ADD", "ff_abadd"));
        rows.add(angleOnlyRow(row, "Calcaneal Inv", "calc_inv"));
        rows.add(angleOnlyRow(row, "Calcaneal Ever", "calc_ever"));
        rows.add(new Row("Leg Length (cm)", Arrays.asList(
                PtEvaluationFormat.plainNumberCell(f(row, "leg_length_r")),
                PtEvaluationFormat.plainNumberCell(f(row, "leg_length_l")),
                null, null)));
        rows.add(new Row("Knee Varus/Valgus", Arrays.asList(
                PtEvaluationFormat.signedAngleCell(i(row, "q_angle_r"), s(row, "q_angle_r_interp")),
                PtEvaluationFormat.signedAngleCell(i(row, "q_angle_l"), s(row, "q_angle_l_interp")),
                null, null)));

        return Section.table(null, columnGroups, columns, rows, null);
    }

    /** hip_flex-style row: integer PROM angle on both sides, plus a text strength grade pair. */
    private Row angleRow(Map<String, Object> row, String label, String prefix) {
        String right = PtEvaluationFormat.angleCell(i(row, prefix + "_r"), s(row, prefix + "_r_interp"));
        String left = PtEvaluationFormat.angleCell(i(row, prefix + "_l"), s(row, prefix + "_l_interp"));
        String strengthRight = PtEvaluationFormat.gradeCell(s(row, prefix + "_s_r"), s(row, prefix + "_s_r_interp"));
        String strengthLeft = PtEvaluationFormat.gradeCell(s(row, prefix + "_s_l"), s(row, prefix + "_s_l_interp"));
        return new Row(label, Arrays.asList(right, left, strengthRight, strengthLeft));
    }

    /** Integer-angle row with no strength columns at all (grey filler cells in the legacy page). */
    private Row angleOnlyRow(Map<String, Object> row, String label, String prefix) {
        return new Row(label, Arrays.asList(
                PtEvaluationFormat.angleCell(i(row, prefix + "_r"), s(row, prefix + "_r_interp")),
                PtEvaluationFormat.angleCell(i(row, prefix + "_l"), s(row, prefix + "_l_interp")),
                null, null));
    }

    /** Text-typed PROM column with no strength pair (TMA/TFA/FF AB-ADD are stored as text). */
    private Row textOnlyRow(Map<String, Object> row, String label, String prefix) {
        return new Row(label, Arrays.asList(
                PtEvaluationFormat.gradeCell(s(row, prefix + "_r"), s(row, prefix + "_r_interp")),
                PtEvaluationFormat.gradeCell(s(row, prefix + "_l"), s(row, prefix + "_l_interp")),
                null, null));
    }

    private Row popAngleRow(Map<String, Object> row) {
        return new Row("Pop angle", Arrays.asList(
                PtEvaluationFormat.rangeCell(i(row, "pop_r_low"), i(row, "pop_r_high"), s(row, "pop_r_interp")),
                PtEvaluationFormat.rangeCell(i(row, "pop_l_low"), i(row, "pop_l_high"), s(row, "pop_l_interp")),
                null, null));
    }

    private Integer i(Map<String, Object> row, String key) {
        return (Integer) row.get(key);
    }

    private String s(Map<String, Object> row, String key) {
        Object value = row.get(key);
        return value == null ? null : value.toString();
    }

    private Float f(Map<String, Object> row, String key) {
        Object value = row.get(key);
        if (value == null) {
            return null;
        }
        return value instanceof Float flt ? flt : ((Number) value).floatValue();
    }

    // --- Phase 2: modify_pt_prom.jsp (the PROM/Strength edit form) ---------------------------

    /** One row of modify_pt_prom.jsp's generic PromRow()-style layout (angle + interp both
     * sides, an optional text strength pair, an optional normals reference range) - everything
     * except Pop angle (two values per side, its own section below), Leg Length (no interp, a
     * float not an int) and Knee Varus/Valgus - aka q_angle (a signed int, legacy prefixes "+"
     * for >= 0 - still parses fine since Integer.parseInt accepts a leading '+').
     *
     * <p>numeric gates the "999 = WNL, blank = not tested" validation legacy's onSubmit actually
     * wires up (MM_validateForm's NisNum list) - EXCEPT tma/tfa/ff_abadd, which that same list
     * also names despite being text columns in data_prom (data_prom.tma_r etc. are text, not
     * integer) - a stale copy-paste in legacy's own validator, not something to port literally.
     *
     * <p>hasNormals pulls the reference range from data_prom_normals as this field's `help` -
     * that table is entirely empty in the current seed data, so help is null for everyone until
     * it's seeded; flagged separately, not bent here.
     */
    /** Package-visible (not private) so ComparisonReportService can reuse the exact same field
     * list rather than re-deriving it from the legacy JSP a second time. */
    record PromField(String label, String prefix, boolean numeric, boolean hasStrength,
            boolean hasNormals) {
    }

    static final List<PromField> PROM_FIELDS = List.of(
            new PromField("Hip Flex", "hip_flex", true, true, true),
            new PromField("Hip Ext", "hip_ext", true, true, true),
            new PromField("Hip Abd", "hip_abd", true, true, true),
            new PromField("Hip Int Rot", "hip_int_rot", true, false, true),
            new PromField("Hip Ext Rot", "hip_ext_rot", true, false, true),
            new PromField("Ryder Test", "ryder_test", true, false, true),
            new PromField("Knee Ext", "knee_ext", true, true, true),
            new PromField("Extensor Lag", "extensor_lag", true, false, true),
            new PromField("Knee Flex", "knee_flex", true, true, true),
            new PromField("Bilateral pop angle", "bilateral_pop", true, false, true),
            new PromField("Ely Test", "ely", true, false, true),
            new PromField("Dorsi (flex)", "dorsi_f", true, true, true),
            new PromField("Dorsi (ext)", "dorsi_e", true, false, true),
            new PromField("Plantar", "plantar", true, true, true),
            new PromField("Ankle Inv", "ankle_inv", true, true, true),
            new PromField("Ankle Ever", "ankle_ever", true, true, true),
            new PromField("TMA", "tma", false, false, false),
            new PromField("TFA", "tfa", false, false, false),
            new PromField("FF AB/ADD", "ff_abadd", false, false, false),
            new PromField("Calcaneal Inv", "calc_inv", true, false, true),
            new PromField("Calcaneal Ever", "calc_ever", true, false, true));

    private static final String WNL_HELP = "999 = Within Normal Limits. Leave blank if not tested.";

    /** modify_pt_prom.jsp's isMovementDisorder field list exactly: hip_abd/knee_ext/pop/dorsi_f
     * (all with a strength pair)/dorsi_e (no strength pair), plus the three Upper Extremity
     * items. hasNormals=false throughout - data_prom_normals is entirely empty right now and,
     * even if seeded, modify_pt_prom.jsp's own MD branch never references it the way the Gait
     * branch does, so there's nothing to port here, not an oversight. */
    static final List<PromField> MD_PROM_FIELDS = List.of(
            new PromField("Hip Abd", "hip_abd", true, true, false),
            new PromField("Knee Ext", "knee_ext", true, true, false),
            new PromField("Dorsi (flex)", "dorsi_f", true, true, false),
            new PromField("Dorsi (ext)", "dorsi_e", true, false, false),
            new PromField("Shoulder Abduction", "shoulder_abduction", true, true, false),
            new PromField("Elbow Extension", "elbow_extension", true, true, false),
            new PromField("Wrist Extension", "wrist_extension", true, true, false));

    public PtEvaluationForm buildForm(int visitId) {
        return buildForm(visitId, false);
    }

    public PtEvaluationForm buildForm(int visitId, boolean isMovementDisorder) {
        if (isMovementDisorder) {
            return buildMdForm(visitId);
        }
        Map<String, Object> row = repository.findPromByVisitId(visitId)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND,
                        "No PROM/Strength record exists yet for this visit."));
        Map<String, Object> normals = repository.findNormalsForVisit(visitId).orElse(Map.of());

        List<FormRow> promRows = new ArrayList<>();
        List<FormRow> strengthRows = new ArrayList<>();
        for (PromField field : PROM_FIELDS) {
            promRows.add(promRow(row, field, normals));
            if (field.hasStrength()) {
                strengthRows.add(strengthRow(row, field));
            }
        }

        FormSection promSection = FormSection.grid("PROM", List.of("Right", "R Interp", "Left", "L Interp"),
                promRows);
        FormSection strengthSection = FormSection.grid("Strength", List.of("Right", "R Interp", "Left", "L Interp"),
                strengthRows);
        FormSection legLengthSection = FormSection.stack("Leg Length (cm)", List.of(
                Field.text("leg_length_r", "Leg Length (cm) Right", numStr(f(row, "leg_length_r"))),
                Field.text("leg_length_l", "Leg Length (cm) Left", numStr(f(row, "leg_length_l")))));
        FormSection qAngleSection = FormSection.grid("Knee Varus/Valgus", List.of("Right", "R Interp", "Left",
                "L Interp"), List.of(qAngleRow(row, normals)));
        FormSection popAngleSection = FormSection.stack("Pop angle", List.of(
                Field.text("pop_r_low", "Pop angle Right low", idStr(i(row, "pop_r_low"))),
                Field.text("pop_r_high", "Pop angle Right high", idStr(i(row, "pop_r_high"))),
                Field.text("pop_r_interp", "Pop angle Right interp", s(row, "pop_r_interp")),
                Field.text("pop_l_low", "Pop angle Left low", idStr(i(row, "pop_l_low"))),
                Field.text("pop_l_high", "Pop angle Left high", idStr(i(row, "pop_l_high"))),
                Field.text("pop_l_interp", "Pop angle Left interp", s(row, "pop_l_interp"))));

        return new PtEvaluationForm("prom-strength", "Modify PROM & Strength", revision.of(row),
                List.of(promSection, strengthSection, popAngleSection, legLengthSection, qAngleSection));
    }

    private PtEvaluationForm buildMdForm(int visitId) {
        Map<String, Object> row = repository.findMdByVisitId(visitId)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND,
                        "No PROM/Strength record exists yet for this visit."));

        List<FormRow> promRows = new ArrayList<>();
        List<FormRow> strengthRows = new ArrayList<>();
        for (PromField field : MD_PROM_FIELDS) {
            promRows.add(promRow(row, field, Map.of()));
            if (field.hasStrength()) {
                strengthRows.add(strengthRow(row, field));
            }
        }

        FormSection promSection = FormSection.grid("PROM", List.of("Right", "R Interp", "Left", "L Interp"),
                promRows);
        FormSection strengthSection = FormSection.grid("Strength", List.of("Right", "R Interp", "Left", "L Interp"),
                strengthRows);
        FormSection popAngleSection = FormSection.stack("Pop angle", List.of(
                Field.text("pop_r_low", "Pop angle Right low", idStr(i(row, "pop_r_low"))),
                Field.text("pop_r_high", "Pop angle Right high", idStr(i(row, "pop_r_high"))),
                Field.text("pop_r_interp", "Pop angle Right interp", s(row, "pop_r_interp")),
                Field.text("pop_l_low", "Pop angle Left low", idStr(i(row, "pop_l_low"))),
                Field.text("pop_l_high", "Pop angle Left high", idStr(i(row, "pop_l_high"))),
                Field.text("pop_l_interp", "Pop angle Left interp", s(row, "pop_l_interp"))));

        return new PtEvaluationForm("prom-strength", "Modify PROM & Strength", revision.of(row),
                List.of(promSection, strengthSection, popAngleSection));
    }

    private FormRow promRow(Map<String, Object> row, PromField field, Map<String, Object> normals) {
        String help = field.hasNormals() ? normalsHelp(normals, field.prefix()) : null;
        Field right = field.numeric()
                ? numberFieldWithHelp(field.prefix() + "_r", field.label() + " Right", idStr(i(row, field.prefix() + "_r")), help)
                : Field.text(field.prefix() + "_r", field.label() + " Right", s(row, field.prefix() + "_r"));
        Field rightInterp = Field.text(field.prefix() + "_r_interp", field.label() + " R Interp",
                s(row, field.prefix() + "_r_interp"));
        Field left = field.numeric()
                ? numberFieldWithHelp(field.prefix() + "_l", field.label() + " Left", idStr(i(row, field.prefix() + "_l")), help)
                : Field.text(field.prefix() + "_l", field.label() + " Left", s(row, field.prefix() + "_l"));
        Field leftInterp = Field.text(field.prefix() + "_l_interp", field.label() + " L Interp",
                s(row, field.prefix() + "_l_interp"));
        return new FormRow(field.label(), Arrays.asList(right, rightInterp, left, leftInterp));
    }

    private FormRow strengthRow(Map<String, Object> row, PromField field) {
        String prefix = field.prefix() + "_s";
        Field right = Field.text(prefix + "_r", field.label() + " Strength Right", s(row, prefix + "_r"));
        Field rightInterp = Field.text(prefix + "_r_interp", field.label() + " Strength R Interp",
                s(row, prefix + "_r_interp"));
        Field left = Field.text(prefix + "_l", field.label() + " Strength Left", s(row, prefix + "_l"));
        Field leftInterp = Field.text(prefix + "_l_interp", field.label() + " Strength L Interp",
                s(row, prefix + "_l_interp"));
        return new FormRow(field.label(), Arrays.asList(right, rightInterp, left, leftInterp));
    }

    private FormRow qAngleRow(Map<String, Object> row, Map<String, Object> normals) {
        String help = normalsHelp(normals, "q_angle");
        Field right = numberFieldWithHelp("q_angle_r", "Knee Varus/Valgus Right",
                signedIdStr(i(row, "q_angle_r")), help);
        Field rightInterp = Field.text("q_angle_r_interp", "Knee Varus/Valgus R Interp", s(row, "q_angle_r_interp"));
        Field left = numberFieldWithHelp("q_angle_l", "Knee Varus/Valgus Left",
                signedIdStr(i(row, "q_angle_l")), help);
        Field leftInterp = Field.text("q_angle_l_interp", "Knee Varus/Valgus L Interp", s(row, "q_angle_l_interp"));
        return new FormRow("Knee Varus/Valgus", Arrays.asList(right, rightInterp, left, leftInterp));
    }

    private static Field numberFieldWithHelp(String name, String label, String value, String help) {
        return new Field(name, label, "number", value, null, false, null, null, null, null,
                help == null ? WNL_HELP : help + ". " + WNL_HELP);
    }

    /** "{mean - sd} - {mean + sd}", one decimal place, matching modify_pt_prom.jsp's own
     * NumberFormat (max 1 fraction digit, no grouping). null when this bucket's row is missing
     * the field's mean/sd pair - true for every field right now, since data_prom_normals has no
     * rows at all in the current seed data. */
    private static String normalsHelp(Map<String, Object> normals, String prefix) {
        Object mean = normals.get(prefix);
        Object sd = normals.get(prefix + "_sd");
        if (!(mean instanceof Number meanNumber) || !(sd instanceof Number sdNumber)) {
            return null;
        }
        float min = meanNumber.floatValue() - sdNumber.floatValue();
        float max = meanNumber.floatValue() + sdNumber.floatValue();
        return "Normal range: " + oneDecimal(min) + " - " + oneDecimal(max);
    }

    private static String oneDecimal(float value) {
        return new java.text.DecimalFormat("0.#").format(value);
    }

    private static String idStr(Integer value) {
        return value == null ? null : String.valueOf(value);
    }

    private static String numStr(Float value) {
        return value == null ? null : String.valueOf(value);
    }

    /** Legacy prefixes a "+" for a non-negative q_angle so the sign is always visible - kept for
     * fidelity; Integer.parseInt happily accepts a leading '+' back on save. */
    private static String signedIdStr(Integer value) {
        if (value == null) {
            return null;
        }
        return value >= 0 ? "+" + value : String.valueOf(value);
    }

    @Transactional
    public PtEvaluationTab save(int patientId, int visitId, String submittedRevision, Map<String, Object> values) {
        return save(patientId, visitId, submittedRevision, values, false);
    }

    @Transactional
    public PtEvaluationTab save(int patientId, int visitId, String submittedRevision, Map<String, Object> values,
            boolean isMovementDisorder) {
        if (isMovementDisorder) {
            return saveMd(patientId, visitId, submittedRevision, values);
        }
        Map<String, Object> row = repository.findPromByVisitId(visitId)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND,
                        "No PROM/Strength record exists yet for this visit."));
        if (!revision.of(row).equals(submittedRevision)) {
            throw new ResponseStatusException(HttpStatus.CONFLICT,
                    "This data has changed since you loaded it. Reload and try again.");
        }

        Map<String, String> errors = new LinkedHashMap<>();
        Map<String, Object> columns = new LinkedHashMap<>();

        for (PromField field : PROM_FIELDS) {
            putFieldColumns(columns, errors, values, row, field);
        }

        columns.put("leg_length_r", resolveFloat(values, "leg_length_r", f(row, "leg_length_r"), errors));
        columns.put("leg_length_l", resolveFloat(values, "leg_length_l", f(row, "leg_length_l"), errors));
        putPopAngleColumns(columns, errors, values, row);
        columns.put("q_angle_r", resolveInt(values, "q_angle_r", (Integer) row.get("q_angle_r"), errors));
        columns.put("q_angle_r_interp", resolveText(values, "q_angle_r_interp", s(row, "q_angle_r_interp")));
        columns.put("q_angle_l", resolveInt(values, "q_angle_l", (Integer) row.get("q_angle_l"), errors));
        columns.put("q_angle_l_interp", resolveText(values, "q_angle_l_interp", s(row, "q_angle_l_interp")));

        if (!errors.isEmpty()) {
            throw new PtEvaluationValidationException("Some PROM/Strength fields are invalid.", errors);
        }

        repository.update((Integer) row.get("id"), columns);
        return build(patientId, visitId, "prom-strength");
    }

    private PtEvaluationTab saveMd(int patientId, int visitId, String submittedRevision,
            Map<String, Object> values) {
        Map<String, Object> row = repository.findMdByVisitId(visitId)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND,
                        "No PROM/Strength record exists yet for this visit."));
        if (!revision.of(row).equals(submittedRevision)) {
            throw new ResponseStatusException(HttpStatus.CONFLICT,
                    "This data has changed since you loaded it. Reload and try again.");
        }

        Map<String, String> errors = new LinkedHashMap<>();
        Map<String, Object> columns = new LinkedHashMap<>();
        for (PromField field : MD_PROM_FIELDS) {
            putFieldColumns(columns, errors, values, row, field);
        }
        putPopAngleColumns(columns, errors, values, row);

        if (!errors.isEmpty()) {
            throw new PtEvaluationValidationException("Some PROM/Strength fields are invalid.", errors);
        }

        repository.updateMd((Integer) row.get("id"), columns);
        return build(patientId, visitId, "prom-strength", true);
    }

    private void putFieldColumns(Map<String, Object> columns, Map<String, String> errors, Map<String, Object> values,
            Map<String, Object> row, PromField field) {
        if (field.numeric()) {
            columns.put(field.prefix() + "_r", resolveInt(values, field.prefix() + "_r", (Integer) row.get(field.prefix() + "_r"), errors));
            columns.put(field.prefix() + "_l", resolveInt(values, field.prefix() + "_l", (Integer) row.get(field.prefix() + "_l"), errors));
        } else {
            columns.put(field.prefix() + "_r", resolveText(values, field.prefix() + "_r", s(row, field.prefix() + "_r")));
            columns.put(field.prefix() + "_l", resolveText(values, field.prefix() + "_l", s(row, field.prefix() + "_l")));
        }
        columns.put(field.prefix() + "_r_interp",
                resolveText(values, field.prefix() + "_r_interp", s(row, field.prefix() + "_r_interp")));
        columns.put(field.prefix() + "_l_interp",
                resolveText(values, field.prefix() + "_l_interp", s(row, field.prefix() + "_l_interp")));
        if (field.hasStrength()) {
            String sPrefix = field.prefix() + "_s";
            columns.put(sPrefix + "_r", resolveText(values, sPrefix + "_r", s(row, sPrefix + "_r")));
            columns.put(sPrefix + "_r_interp", resolveText(values, sPrefix + "_r_interp", s(row, sPrefix + "_r_interp")));
            columns.put(sPrefix + "_l", resolveText(values, sPrefix + "_l", s(row, sPrefix + "_l")));
            columns.put(sPrefix + "_l_interp", resolveText(values, sPrefix + "_l_interp", s(row, sPrefix + "_l_interp")));
        }
    }

    private void putPopAngleColumns(Map<String, Object> columns, Map<String, String> errors,
            Map<String, Object> values, Map<String, Object> row) {
        columns.put("pop_r_low", resolveInt(values, "pop_r_low", (Integer) row.get("pop_r_low"), errors));
        columns.put("pop_r_high", resolveInt(values, "pop_r_high", (Integer) row.get("pop_r_high"), errors));
        columns.put("pop_r_interp", resolveText(values, "pop_r_interp", s(row, "pop_r_interp")));
        columns.put("pop_l_low", resolveInt(values, "pop_l_low", (Integer) row.get("pop_l_low"), errors));
        columns.put("pop_l_high", resolveInt(values, "pop_l_high", (Integer) row.get("pop_l_high"), errors));
        columns.put("pop_l_interp", resolveText(values, "pop_l_interp", s(row, "pop_l_interp")));
    }

    private static String resolveText(Map<String, Object> values, String key, String currentValue) {
        if (!values.containsKey(key)) {
            return currentValue;
        }
        Object raw = values.get(key);
        String submitted = raw == null ? null : raw.toString();
        return (submitted == null || submitted.isEmpty()) ? null : submitted;
    }

    private static Integer resolveInt(Map<String, Object> values, String key, Integer currentValue,
            Map<String, String> errors) {
        if (!values.containsKey(key)) {
            return currentValue;
        }
        Object raw = values.get(key);
        String submitted = raw == null ? null : raw.toString();
        if (submitted == null || submitted.isEmpty()) {
            return null;
        }
        try {
            return Integer.parseInt(submitted);
        } catch (NumberFormatException e) {
            errors.put(key, "Must be a whole number (999 = Within Normal Limits).");
            return currentValue;
        }
    }

    private static Float resolveFloat(Map<String, Object> values, String key, Float currentValue,
            Map<String, String> errors) {
        if (!values.containsKey(key)) {
            return currentValue;
        }
        Object raw = values.get(key);
        String submitted = raw == null ? null : raw.toString();
        if (submitted == null || submitted.isEmpty()) {
            return null;
        }
        try {
            return Float.parseFloat(submitted);
        } catch (NumberFormatException e) {
            errors.put(key, "Must be a number.");
            return currentValue;
        }
    }

    public PtEvaluationTab createRecord(int patientId, int visitId) {
        return createRecord(patientId, visitId, false);
    }

    public PtEvaluationTab createRecord(int patientId, int visitId, boolean isMovementDisorder) {
        if (isMovementDisorder) {
            if (repository.findMdByVisitId(visitId).isPresent()) {
                throw new ResponseStatusException(HttpStatus.CONFLICT,
                        "PROM/Strength data already exists for this visit.");
            }
            repository.insertBlankMd(visitId);
            return build(patientId, visitId, "prom-strength", true);
        }
        if (repository.findPromByVisitId(visitId).isPresent()) {
            throw new ResponseStatusException(HttpStatus.CONFLICT,
                    "PROM/Strength data already exists for this visit.");
        }
        repository.insertBlank(visitId);
        return build(patientId, visitId, "prom-strength");
    }
}













package org.nemours.gaitlab.service;

import org.springframework.security.core.Authentication;

/**
 * Button-visibility flags for the PT Evaluation tabs, mirroring the admin gating on the legacy
 * pt.jsp/pt-eval-*.jsp pages: Create New Record, Modify General and PROM/Strength "Edit Data",
 * and Add an Orthosis (and per-row Orthotics edit/delete) are admin-only; editing Motor
 * Control/Tone/Pain values once a record exists is not. report (the Comparison Report button) is
 * hidden only for movement-disorder visits, regardless of role.
 */
public record PtEvaluationActions(boolean create, boolean edit, boolean add, boolean report) {

    public static boolean isAdmin(Authentication authentication) {
        return authentication != null && authentication.getAuthorities().stream()
                .anyMatch(authority -> authority.getAuthority().equals("ROLE_ADMIN"));
    }

    /** new_visit_type.id for "Movement Disorder" - VisitType.MOVEMENT_DISORDER_ID in the legacy
     * Java source, matching pt-sections.jsp's own isMovementDisorder check. */
    public static final int MOVEMENT_DISORDER_VISIT_TYPE_ID = 11;

    public static boolean isMovementDisorder(Integer visitTypeId) {
        return visitTypeId != null && visitTypeId == MOVEMENT_DISORDER_VISIT_TYPE_ID;
    }
}












package org.nemours.gaitlab.service;

import java.util.List;

/** Shared response shape for the PT Evaluation edit-form endpoints (GET .../{tab}/form), matching
 * the frontend's generic form renderer contract. */
public record PtEvaluationForm(String tab, String title, String revision, List<FormSection> sections) {

    public record FormRow(String label, List<Field> fields) {
    }

    public record FormSection(String title, String layout, List<String> columns, List<FormRow> rows,
            List<Field> fields) {

        public static FormSection grid(String title, List<String> columns, List<FormRow> rows) {
            return new FormSection(title, "grid", columns, rows, null);
        }

        public static FormSection stack(String title, List<Field> fields) {
            return new FormSection(title, "stack", null, null, fields);
        }
    }

    public record Option(String value, String label) {
    }

    /** value is a String for every scalar field type, or a List<String> for multiselect. */
    public record Field(String name, String label, String type, Object value, List<Option> options,
            boolean required, Double min, Double max, Double step, Integer maxLength, String help) {

        public static Field text(String name, String label, String value) {
            return new Field(name, label, "text", value, null, false, null, null, null, null, null);
        }

        public static Field textarea(String name, String label, String value) {
            return new Field(name, label, "textarea", value, null, false, null, null, null, null, null);
        }

        public static Field select(String name, String label, String value, List<Option> options) {
            return new Field(name, label, "select", value, options, false, null, null, null, null, null);
        }

        public static Field radio(String name, String label, String value, List<Option> options) {
            return new Field(name, label, "radio", value, options, false, null, null, null, null, null);
        }
    }
}













package org.nemours.gaitlab.service;

/**
 * Display-formatting rules shared by the PT evaluation tab ports (pt-eval-prom.jsp and friends).
 * Each legacy page repeats these same ternaries dozens of times; centralizing them here is the
 * only way to port them exactly without 1000 lines of copy-pasted ternaries.
 */
final class PtEvaluationFormat {
    private PtEvaluationFormat() {
    }

    private static final String WNL_SENTINEL = "999";

    /** True when a blank cell (null in the response) should be produced because neither the
     * measurement nor its interpretation has anything to show. */
    private static boolean bothBlank(Object value, String interp) {
        return value == null && isBlank(interp);
    }

    private static boolean isBlank(String s) {
        return s == null || s.isEmpty();
    }

    /** hip_flex_r / hip_flex_r_interp style cell: an integer angle, "999" => "WNL", "value [interp]". */
    static String angleCell(Integer value, String interp) {
        if (bothBlank(value, interp)) {
            return null;
        }
        String v = value == null ? "" : (value == 999 ? "WNL" : String.valueOf(value));
        return v + " [" + (interp == null ? "" : interp) + "]";
    }

    /** Strength-grade style cell: a free-text grade (e.g. "4+"), literal "999" => "WNL". */
    static String gradeCell(String value, String interp) {
        if (bothBlank(value, interp)) {
            return null;
        }
        String v = value == null ? "" : (WNL_SENTINEL.equals(value) ? "WNL" : value);
        return v + " [" + (interp == null ? "" : interp) + "]";
    }

    /** q_angle style cell: "999" => "WNL", otherwise a "+" prefix on non-negative values. */
    static String signedAngleCell(Integer value, String interp) {
        if (bothBlank(value, interp)) {
            return null;
        }
        String v;
        if (value == null) {
            v = "";
        } else if (value == 999) {
            v = "WNL";
        } else {
            v = value >= 0 ? "+" + value : String.valueOf(value);
        }
        return v + " [" + (interp == null ? "" : interp) + "]";
    }

    /** Pop angle style cell: a "low -- high [interp]" range, "999" => "WNL" on the low value only
     * (the legacy page never applies the WNL check to the high value). */
    static String rangeCell(Integer low, Integer high, String interp) {
        if (low == null && high == null && isBlank(interp)) {
            return null;
        }
        String lowStr = low == null ? "" : (low == 999 ? "WNL" : String.valueOf(low));
        String highStr = high == null ? "" : String.valueOf(high);
        return lowStr + " -- " + highStr + " [" + (interp == null ? "" : interp) + "]";
    }

    /** Leg length style cell: a plain real number, no interpretation bracket. The legacy page's
     * "999" check compares a Float's toString() ("999.0") against the literal "999" and so never
     * actually fires for this column; that's not reproduced here since it's a dead rule, not a
     * display format. */
    static String plainNumberCell(Float value) {
        if (value == null) {
            return null;
        }
        if (value == Math.floor(value) && !value.isInfinite()) {
            return String.valueOf(value.intValue());
        }
        return String.valueOf(value);
    }

    /** A plain cell with no formatting rules at all (Motor Control / Tone score and interp cells). */
    static String plainCell(String value) {
        return isBlank(value) ? null : value;
    }
}












package org.nemours.gaitlab.service;

import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.HexFormat;

import org.springframework.stereotype.Component;

import tools.jackson.databind.ObjectMapper;

/**
 * Opaque optimistic-concurrency token for the PT Evaluation edit forms: a SHA-256 hash of the
 * current view, the same scheme VideoAssessmentService/QuestionnaireResponseService already use
 * elsewhere in this app. Legacy itself has no such check at all (a raw unconditional UPDATE) -
 * this is a deliberate addition per the write-endpoint spec, not a legacy behavior to match.
 */
@Component
public class PtEvaluationRevision {
    private final ObjectMapper json;

    public PtEvaluationRevision(ObjectMapper json) {
        this.json = json;
    }

    public String of(Object value) {
        try {
            return HexFormat.of().formatHex(MessageDigest.getInstance("SHA-256")
                    .digest(json.writeValueAsString(value).getBytes(StandardCharsets.UTF_8)));
        } catch (NoSuchAlgorithmException e) {
            throw new IllegalStateException(e);
        }
    }
}














package org.nemours.gaitlab.service;

import java.text.DecimalFormat;
import java.time.LocalDate;
import java.time.temporal.ChronoUnit;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.Map;

import org.nemours.gaitlab.repository.PtEvaluationRepository;
import org.nemours.gaitlab.repository.VisitRepository;
import org.nemours.gaitlab.repository.VisitRepository.GmfcsListEntry;
import org.nemours.gaitlab.repository.VisitRepository.PtGeneralEvaluationRow;
import org.nemours.gaitlab.service.PtEvaluationForm.Field;
import org.nemours.gaitlab.service.PtEvaluationForm.FormRow;
import org.nemours.gaitlab.service.PtEvaluationForm.FormSection;
import org.nemours.gaitlab.service.PtEvaluationForm.Option;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.server.ResponseStatusException;

/**
 * Ports lab/data/pt.jsp (the "General & GMFM" PT evaluation tab) for the non-movement-disorder
 * case. Movement-disorder visits (MACS/CFCS, data_pt_movement_disorder) are not covered here.
 */
@Service
public class PtEvaluationService {

    public record General(
            Float heightCm, Float weightKg, String heightZ, String weightZ,
            Float rightFootLengthCm, Float leftFootLengthCm,
            Float rightFootWidthCm, Float leftFootWidthCm,
            String ambulationScale,
            String gmfcs,
            String fms5m, String fms50m, String fms500m,
            String comments) {
    }

    public record GmfmTask(String items, String task, boolean bilateral, Integer right, Integer left) {
    }

    public record GmfmModule(String module, Integer score, Integer maxScore, Integer percent) {
    }

    public record GeneralEvaluation(
            Integer patientId, Integer visitId,
            General general,
            List<GmfmTask> gmfmD,
            List<GmfmModule> gmfmModules,
            Double meanPercent,
            PtEvaluationActions actions) {

        public GeneralEvaluation withActions(PtEvaluationActions actions) {
            return new GeneralEvaluation(patientId, visitId, general, gmfmD, gmfmModules, meanPercent, actions);
        }
    }

    private final VisitRepository visits;
    private final PtEvaluationRepository general;
    private final PtEvaluationRevision revision;

    public PtEvaluationService(VisitRepository visits, PtEvaluationRepository general, PtEvaluationRevision revision) {
        this.visits = visits;
        this.general = general;
        this.revision = revision;
    }

    public GeneralEvaluation getGeneral(int patientId, int visitId) {
        visits.findInterpretationVisitByIdAndPatientId(visitId, patientId)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));

        return visits.findPtGeneralEvaluationByVisitIdAndPatientId(visitId, patientId)
                .map(row -> buildEvaluation(patientId, visitId, row))
                .orElseGet(() -> emptyEvaluation(patientId, visitId));
    }

    private GeneralEvaluation emptyEvaluation(int patientId, int visitId) {
        General general = new General(
                null, null, null, null, null, null, null, null, null, null, null, null, null, null);
        return new GeneralEvaluation(patientId, visitId, general, List.of(), List.of(), null, null);
    }

    private GeneralEvaluation buildEvaluation(int patientId, int visitId, PtGeneralEvaluationRow row) {
        int pullStand = z(row.getPullStand());
        int indepStand3 = z(row.getIndepStand3());
        int indepStand20 = z(row.getIndepStand20());
        int oneLegHandR = z(row.getOneLegHandR());
        int oneLegHandL = z(row.getOneLegHandL());
        int oneLegR = z(row.getOneLegR());
        int oneLegL = z(row.getOneLegL());
        int shortSitStand = z(row.getShortSitStand());
        int highKneeStandR = z(row.getHighKneeStandR());
        int highKneeStandL = z(row.getHighKneeStandL());
        int lowerFloor = z(row.getLowerFloor());
        int squats = z(row.getSquats());
        int standPickPen = z(row.getStandPickPen());

        List<GmfmTask> gmfmD = List.of(
                new GmfmTask("52", "Pull to stand", false, pullStand, null),
                new GmfmTask("53", "Indep. Standing - Arms Free (3 sec)", false, indepStand3, null),
                new GmfmTask("56", "Indep. Standing - Arms Free (20 sec)", false, indepStand20, null),
                new GmfmTask("54,55", "One leg stance hand held - Right", true, oneLegHandR, oneLegHandL),
                new GmfmTask("57,58", "One leg stance", true, oneLegR, oneLegL),
                new GmfmTask("59", "Short sit to stand", false, shortSitStand, null),
                new GmfmTask("60,61", "High kneel to stand", true, highKneeStandR, highKneeStandL),
                new GmfmTask("62", "Lowers to floor", false, lowerFloor, null),
                new GmfmTask("63", "Squats", false, squats, null),
                new GmfmTask("64", "Standing - picks pen from floor", false, standPickPen, null));

        // Module D is always scored from the GMFM-D task total; A, B, C, E only when their raw
        // column is non-null and non-zero, matching the legacy page's "if (curr_mod)" JS check -
        // a raw score of exactly 0 is indistinguishable from "no entry" there, and this preserves
        // that quirk rather than silently fixing it.
        int moduleDTotal = pullStand + indepStand3 + indepStand20 + oneLegHandR + oneLegHandL
                + oneLegR + oneLegL + shortSitStand + highKneeStandR + highKneeStandL
                + lowerFloor + squats + standPickPen;

        General general = new General(
                row.getHeightCm(), row.getWeightKg(),
                heightZ(row.getHeightCm(), row.getGender(), row.getVisitDate(), row.getDob()),
                weightZ(row.getWeightKg(), row.getGender(), row.getVisitDate(), row.getDob()),
                row.getRightFootLengthCm(), row.getLeftFootLengthCm(),
                row.getRightFootWidthCm(), row.getLeftFootWidthCm(),
                labelDash(row.getAmbScaleId() == null ? null : String.valueOf(row.getAmbScaleId()),
                        row.getAmbulationScale()),
                gmfcsDisplay(row, pullStand, indepStand3, oneLegR, oneLegL, shortSitStand,
                        highKneeStandR, highKneeStandL, moduleDTotal),
                labelDash(row.getFms5m(), row.getFms5mDesc()),
                labelDash(row.getFms50m(), row.getFms50mDesc()),
                labelDash(row.getFms500m(), row.getFms500mDesc()),
                row.getComments());

        List<GmfmModule> gmfmModules = new ArrayList<>();
        gmfmModules.add(scoredModule("A", row.getGmfmModA(), 51));
        gmfmModules.add(scoredModule("B", row.getGmfmModB(), 60));
        gmfmModules.add(scoredModule("C", row.getGmfmModC(), 42));
        gmfmModules.add(new GmfmModule("D", moduleDTotal, 39, percentOf(moduleDTotal, 39)));
        gmfmModules.add(scoredModule("E", row.getGmfmModE(), 72));

        int runningTotal = 0;
        int enteredModules = 0;
        for (GmfmModule module : gmfmModules) {
            if (module.percent() != null) {
                runningTotal += module.percent();
                enteredModules++;
            }
        }
        Double meanPercent = enteredModules == 0 ? null : (double) runningTotal / enteredModules;

        return new GeneralEvaluation(patientId, visitId, general, gmfmD, List.copyOf(gmfmModules), meanPercent, null);
    }

    /** "{value} -- {description}", legacy's exact separator for Ambulation Scale/FMS/GMFCS -
     * " -- " with both sides blank (never null) when nothing is on file. */
    private static String labelDash(String value, String description) {
        return (value == null ? "" : value) + " -- " + (description == null ? "" : description);
    }

    private static final DecimalFormat Z_FORMAT_PATTERN = new DecimalFormat("0.0");

    private static String zFormat(double value) {
        synchronized (Z_FORMAT_PATTERN) {
            return Z_FORMAT_PATTERN.format(value);
        }
    }

    /** Ported from pt.jsp's height z-score block: growth-curve mean/std by gender, against age
     * in whole months (SQL ROUND((date - dob) / 365.0 * 12), applied before the formula, not
     * after). null when height itself isn't on file - unlike weight, an unknown/other gender
     * prints "need gender" rather than "N/A". */
    private String heightZ(Float heightCm, String gender, LocalDate visitDate, LocalDate dob) {
        if (heightCm == null) {
            return null;
        }
        if (!"Male".equals(gender) && !"Female".equals(gender)) {
            return "(z = need gender)";
        }
        if (visitDate == null || dob == null) {
            return null;
        }
        double months = ageInMonths(visitDate, dob);
        double mean;
        double std;
        if ("Male".equals(gender)) {
            mean = -260.3657 + (262.8692 - (-260.3657)) / (1.0 + 0.57774 * Math.exp(-(0.0052597 * months)));
            std = 2.02485 + (9.8984 - 2.02485) / (1.0 + 7.5227 * Math.exp(-(0.0181343 * months)));
        } else {
            mean = -82.788774 + (195.26046 - (-82.798774)) / (1.0 + 0.8455536 * Math.exp(-(0.0101596 * months)));
            std = 2.40482737 + (6.81956506 - 2.40482737) / (1.0 + 15.723074 * Math.exp(-(0.0450524084 * months)));
        }
        return "(z = " + zFormat((heightCm - mean) / std) + ")";
    }

    /** Ported from pt.jsp's weight z-score block - "N/A" (not "need gender") for an unknown or
     * other gender, matching the legacy page's inconsistency between the two fields. */
    private String weightZ(Float weightKg, String gender, LocalDate visitDate, LocalDate dob) {
        if (weightKg == null) {
            return null;
        }
        if (!"Male".equals(gender) && !"Female".equals(gender)) {
            return "(z = N/A)";
        }
        if (visitDate == null || dob == null) {
            return null;
        }
        double months = ageInMonths(visitDate, dob);
        double mean;
        double std;
        if ("Male".equals(gender)) {
            mean = 3.7 + (146.93 - 3.736) / (1.0 + 21.068 * Math.exp(-(0.013832 * months)));
            std = 0.872199 + (13.9084 - 0.872199) / (1.0 + 59.325 * Math.exp(-(0.0338 * months)));
        } else {
            mean = 6.4032907948 + (74.129378344 - 6.4032907947) / (1.0 + 19.0375155389 * Math.exp(-(0.020912892101 * months)));
            std = 1.16994385 + (12.9379166 - 1.1699438) / (1.0 + 88.52793975 * Math.exp(-(0.041292875 * months)));
        }
        return "(z = " + zFormat((weightKg - mean) / std) + ")";
    }

    /** SQL ROUND((date - dob) / 365.0 * 12) - rounded to whole months BEFORE it feeds the growth
     * formulas, matching pt.jsp's pt_age query exactly (not the unrounded day count). */
    private static double ageInMonths(LocalDate visitDate, LocalDate dob) {
        long days = ChronoUnit.DAYS.between(dob, visitDate);
        return Math.round(days / 365.0 * 12);
    }

    private GmfmModule scoredModule(String module, Integer rawScore, int maxScore) {
        if (rawScore == null || rawScore == 0) {
            return new GmfmModule(module, null, null, null);
        }
        return new GmfmModule(module, rawScore, maxScore, percentOf(rawScore, maxScore));
    }

    private int percentOf(int score, int maxScore) {
        return Math.round(score * 100f / maxScore);
    }

    private int z(Integer value) {
        return value == null ? 0 : value;
    }

    /**
     * Ports the GMFCS row shown next to Ambulation Scale: gated on the diagnosis being CP
     * (patient.gen_diag or visit.icd9), using a manually-recorded gmfcs (data_pt.gmfcs != the
     * unset-sentinel id 1) when present, otherwise the calculated GMFCS from amb_scale/age/GMFM-D
     * scores. null only when the row isn't shown at all (not CP) - every other CP-gated outcome
     * is a display string, "--" (bare, no separator) when nothing can be resolved, matching the
     * legacy page's literal " -- " cell rather than an empty value.
     */
    private String gmfcsDisplay(PtGeneralEvaluationRow row, int gmfm52, int gmfm53, int gmfm57, int gmfm58,
            int gmfm59, int gmfm60, int gmfm61, int gmfmDTotal) {
        if (!isCP(row.getGenDiag()) && !isCP(row.getIcd9())) {
            return null;
        }

        Integer gmfcsId = row.getGmfcsId();
        if (gmfcsId != null && gmfcsId != 1) {
            return gmfcsListEntries().stream()
                    .filter(entry -> gmfcsId.equals(entry.getId()))
                    .findFirst()
                    .map(entry -> labelDash(entry.getLabel(), entry.getDescription()))
                    .orElse("--");
        }

        LocalDate visitDate = row.getVisitDate();
        LocalDate dob = row.getDob();
        if (visitDate == null || dob == null || gmfmDTotal == 0) {
            return "--";
        }
        double age = ChronoUnit.DAYS.between(dob, visitDate) / 365.0;
        String calculatedLabel = calculateGMFCS(z(row.getAmbScaleId()), age, gmfm52, gmfm53, gmfm57, gmfm58, gmfm59,
                gmfm60, gmfm61);
        if (calculatedLabel.isEmpty()) {
            return "--";
        }
        return gmfcsListEntries().stream()
                .filter(entry -> calculatedLabel.equals(entry.getLabel()))
                .findFirst()
                .map(entry -> labelDash(calculatedLabel, entry.getDescription()))
                .orElse("--");
    }

    private List<GmfcsListEntry> gmfcsListEntries() {
        return visits.findAllGmfcsListEntries();
    }

    private boolean isCP(Integer icd9Id) {
        return icd9Id != null && ((icd9Id >= 13 && icd9Id <= 17) || icd9Id == 119 || icd9Id == 142);
    }

    /**
     * Ported verbatim from edu.udel.aidi.gait.questionnaire.util.GMFCSHelper#calculateGMFCS -
     * the struts app's own javadoc calls this "the most recent/correct version" of the GMFCS
     * calculation, matching the data_pt_calculated_gmfcs DB view.
     */
    private String calculateGMFCS(int ambScale, double age, int gmfm52, int gmfm53, int gmfm57, int gmfm58,
            int gmfm59, int gmfm60, int gmfm61) {
        if (ambScale == 1) {
            if ((age < 4.0) || ((age >= 4.0) && (age < 12.0) && (gmfm59 > 2) && (gmfm60 > 2) && (gmfm61 > 2))) {
                return "I";
            } else if ((age >= 4.0) && (age < 12.0) && ((gmfm59 <= 2) || (gmfm60 <= 2) || (gmfm61 <= 2))) {
                return "II";
            } else if ((age >= 12.0) && (age < 18.0) && (gmfm59 > 2) && ((gmfm60 > 2) || (gmfm61 > 2))) {
                return "I";
            } else if ((age >= 12.0) && (age < 18.0) && (gmfm59 <= 2) && (gmfm60 <= 2) && (gmfm61 <= 2)) {
                return "II";
            }
        } else if (ambScale == 2) {
            if (age < 4.0) {
                return "II";
            } else if (age >= 4.0 && age < 6.0 && gmfm59 >= 3 && gmfm60 >= 3 && gmfm61 >= 3 && gmfm57 >= 2
                    && gmfm58 >= 2) {
                return "I";
            } else if (age >= 4.0 && age < 6.0 && gmfm60 <= 2 && gmfm61 <= 2) {
                return "II";
            } else if (age >= 6.0 && age < 12.0 && gmfm59 >= 3 && (gmfm57 >= 2 || gmfm58 >= 2)) {
                return "II";
            } else if (age >= 6.0 && age < 12.0 && gmfm59 <= 2 && (gmfm57 < 2 || gmfm58 < 2)) {
                return "III";
            } else if (age >= 12.0 && gmfm60 >= 2 && gmfm61 >= 2) {
                return "II";
            } else if (age >= 12.0 && gmfm60 <= 1 && gmfm61 <= 1) {
                return "III";
            }
        } else if (ambScale == 3) {
            if (age >= 4.0 && age < 6.0) {
                return "III";
            } else if (age >= 6.0 && age < 12.0 && gmfm52 >= 3) {
                return "III";
            } else if (age >= 6.0 && age < 12.0 && gmfm52 <= 2) {
                return "IV";
            } else if (age >= 12.0 && gmfm59 >= 1 && gmfm60 >= 1 && gmfm61 >= 1) {
                return "III";
            } else if (age >= 12.0 && (gmfm60 < 1 || gmfm61 < 1)) {
                return "IV";
            }
        } else if (ambScale == 4) {
            if (gmfm59 >= 1 && gmfm52 >= 2) {
                return "III";
            } else if (gmfm59 < 1 && gmfm52 <= 1) {
                return "IV";
            }
        } else if (ambScale == 5) {
            if (gmfm52 >= 1 && gmfm53 >= 1) {
                return "IV";
            } else if (gmfm52 == 0 || gmfm53 == 0) {
                return "V";
            }
        } else if (ambScale == 6) {
            return "V";
        }
        return "";
    }

    // --- Phase 2: modify_pt.jsp (the General & GMFM edit form) ---------------------------------

    private static final List<Option> AMB_SCALE_OPTIONS = List.of(
            new Option("1", "1"), new Option("2", "2"), new Option("3", "3"),
            new Option("4", "4"), new Option("5", "5"), new Option("6", "6"));

    public PtEvaluationForm buildForm(int patientId, int visitId) {
        PtGeneralEvaluationRow row = visits.findPtGeneralEvaluationByVisitIdAndPatientId(visitId, patientId)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND,
                        "No general record exists yet for this visit."));

        Field height = Field.text("height", "Height (cm)", numStr(row.getHeightCm()));
        Field weight = Field.text("weight", "Weight (kg)", numStr(row.getWeightKg()));
        Field rFootLength = Field.text("r_foot_length", "Right Foot Length (cm)", numStr(row.getRightFootLengthCm()));
        Field lFootLength = Field.text("l_foot_length", "Left Foot Length (cm)", numStr(row.getLeftFootLengthCm()));
        Field rFootWidth = Field.text("r_foot_width", "Right Foot Width (cm)", numStr(row.getRightFootWidthCm()));
        Field lFootWidth = Field.text("l_foot_width", "Left Foot Width (cm)", numStr(row.getLeftFootWidthCm()));
        Field ambScale = new Field("amb_scale", "Ambulation Scale", "radio", idStr(row.getAmbScaleId()),
                AMB_SCALE_OPTIONS, false, null, null, null, null, null);
        Field gmfcs = Field.select("gmfcs", "GMFCS", idStr(row.getGmfcsId()),
                toIdLabelOptions(general.findActiveGmfcsListEntries()));
        Field fms5 = Field.select("fms_5", "FMS (5m)", idStr(row.getFms5Id()),
                toIdLabelOptions(general.findActiveFmsListEntries()));
        Field fms50 = Field.select("fms_50", "FMS (50m)", idStr(row.getFms50Id()),
                toIdLabelOptions(general.findActiveFmsListEntries()));
        Field fms500 = Field.select("fms_500", "FMS (500m)", idStr(row.getFms500Id()),
                toIdLabelOptions(general.findActiveFmsListEntries()));
        Field comments = Field.textarea("comments", "Comments", row.getComments());

        FormSection generalSection = FormSection.stack("General and Marker Information", List.of(
                height, weight, rFootLength, lFootLength, rFootWidth, lFootWidth, ambScale, gmfcs, fms5, fms50,
                fms500, comments));

        FormSection gmfmDSection = FormSection.grid("Functional Strength -GMFM-D", List.of("Right", "Left"), List.of(
                bilateralRow("Pull to stand (52)", "pull_stand", row.getPullStand(), null, null),
                bilateralRow("Indep. Standing - Arms Free (3 sec) (53)", "indep_stand_3", row.getIndepStand3(),
                        null, null),
                bilateralRow("Indep. Standing - Arms Free (20 sec) (56)", "indep_stand_20", row.getIndepStand20(),
                        null, null),
                bilateralRow("One leg stance hand held - Right (54,55)", "one_leg_hand_r", row.getOneLegHandR(),
                        "one_leg_hand_l", row.getOneLegHandL()),
                bilateralRow("One leg stance (57,58)", "one_leg_r", row.getOneLegR(), "one_leg_l", row.getOneLegL()),
                bilateralRow("Short sit to stand (59)", "short_sit_stand", row.getShortSitStand(), null, null),
                bilateralRow("High kneel to stand (60,61)", "high_knee_stand_r", row.getHighKneeStandR(),
                        "high_knee_stand_l", row.getHighKneeStandL()),
                bilateralRow("Lowers to floor (62)", "lower_floor", row.getLowerFloor(), null, null),
                bilateralRow("Squats (63)", "squats", row.getSquats(), null, null),
                bilateralRow("Standing - picks pen from floor (64)", "stand_pick_pen", row.getStandPickPen(), null,
                        null)));

        FormSection modulesSection = FormSection.stack("Other GMFM Modules", List.of(
                Field.text("gmfm_mod_a", "Module A", idStr(row.getGmfmModA())),
                Field.text("gmfm_mod_b", "Module B", idStr(row.getGmfmModB())),
                Field.text("gmfm_mod_c", "Module C", idStr(row.getGmfmModC())),
                Field.text("gmfm_mod_e", "Module E", idStr(row.getGmfmModE()))));

        return new PtEvaluationForm("general", "Modify General & GMFM", revision.of(formSnapshot(row)),
                List.of(generalSection, gmfmDSection, modulesSection));
    }

    /** A row with a Right-only field (null name/value for Left) or both sides, matching the
     * GMFM-D table's mix of unilateral and bilateral tasks. */
    private static FormRow bilateralRow(String label, String nameR, Integer valueR, String nameL, Integer valueL) {
        Field right = Field.text(nameR, label + " Right", idStr(valueR));
        Field left = nameL == null ? null : Field.text(nameL, label + " Left", idStr(valueL));
        return new FormRow(label, Arrays.asList(right, left));
    }

    private static List<Option> toIdLabelOptions(List<Map<String, Object>> rows) {
        return rows.stream().map(row -> new Option(String.valueOf(row.get("id")), (String) row.get("label")))
                .toList();
    }

    private static String idStr(Integer value) {
        return value == null ? null : String.valueOf(value);
    }

    private static String numStr(Float value) {
        return value == null ? null : String.valueOf(value);
    }

    /** A stable snapshot of everything the form can edit, for the revision hash - the full
     * PtGeneralEvaluationRow also carries read-only context (dob, icd9, gender, labels) that
     * isn't part of what's being saved. */
    private static Map<String, Object> formSnapshot(PtGeneralEvaluationRow row) {
        Map<String, Object> snapshot = new java.util.LinkedHashMap<>();
        snapshot.put("heightCm", row.getHeightCm());
        snapshot.put("weightKg", row.getWeightKg());
        snapshot.put("rightFootLengthCm", row.getRightFootLengthCm());
        snapshot.put("leftFootLengthCm", row.getLeftFootLengthCm());
        snapshot.put("rightFootWidthCm", row.getRightFootWidthCm());
        snapshot.put("leftFootWidthCm", row.getLeftFootWidthCm());
        snapshot.put("ambScaleId", row.getAmbScaleId());
        snapshot.put("gmfcsId", row.getGmfcsId());
        snapshot.put("fms5Id", row.getFms5Id());
        snapshot.put("fms50Id", row.getFms50Id());
        snapshot.put("fms500Id", row.getFms500Id());
        snapshot.put("comments", row.getComments());
        snapshot.put("pullStand", row.getPullStand());
        snapshot.put("indepStand3", row.getIndepStand3());
        snapshot.put("indepStand20", row.getIndepStand20());
        snapshot.put("oneLegHandR", row.getOneLegHandR());
        snapshot.put("oneLegHandL", row.getOneLegHandL());
        snapshot.put("oneLegR", row.getOneLegR());
        snapshot.put("oneLegL", row.getOneLegL());
        snapshot.put("shortSitStand", row.getShortSitStand());
        snapshot.put("highKneeStandR", row.getHighKneeStandR());
        snapshot.put("highKneeStandL", row.getHighKneeStandL());
        snapshot.put("lowerFloor", row.getLowerFloor());
        snapshot.put("squats", row.getSquats());
        snapshot.put("standPickPen", row.getStandPickPen());
        snapshot.put("gmfmModA", row.getGmfmModA());
        snapshot.put("gmfmModB", row.getGmfmModB());
        snapshot.put("gmfmModC", row.getGmfmModC());
        snapshot.put("gmfmModE", row.getGmfmModE());
        return snapshot;
    }

    @Transactional
    public GeneralEvaluation save(int patientId, int visitId, String submittedRevision, Map<String, Object> values) {
        PtGeneralEvaluationRow row = visits.findPtGeneralEvaluationByVisitIdAndPatientId(visitId, patientId)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND,
                        "No general record exists yet for this visit."));

        if (!revision.of(formSnapshot(row)).equals(submittedRevision)) {
            throw new ResponseStatusException(HttpStatus.CONFLICT,
                    "This data has changed since you loaded it. Reload and try again.");
        }

        Map<String, String> errors = new java.util.LinkedHashMap<>();
        Float height = resolveFloat(values, "height", row.getHeightCm(), errors);
        Float weight = resolveFloat(values, "weight", row.getWeightKg(), errors);
        Float rFootLength = resolveFloat(values, "r_foot_length", row.getRightFootLengthCm(), errors);
        Float lFootLength = resolveFloat(values, "l_foot_length", row.getLeftFootLengthCm(), errors);
        Float rFootWidth = resolveFloat(values, "r_foot_width", row.getRightFootWidthCm(), errors);
        Float lFootWidth = resolveFloat(values, "l_foot_width", row.getLeftFootWidthCm(), errors);
        Integer ambScale = resolveFromSet(values, "amb_scale", row.getAmbScaleId(),
                java.util.Set.of(1, 2, 3, 4, 5, 6), errors);
        Integer gmfcs = resolveLookupId(values, "gmfcs", row.getGmfcsId(), general::existsGmfcsListEntry, errors);
        Integer fms5 = resolveLookupId(values, "fms_5", row.getFms5Id(), general::existsFmsListEntry, errors);
        Integer fms50 = resolveLookupId(values, "fms_50", row.getFms50Id(), general::existsFmsListEntry, errors);
        Integer fms500 = resolveLookupId(values, "fms_500", row.getFms500Id(), general::existsFmsListEntry, errors);
        String comments = resolveText(values, "comments", row.getComments());
        Integer pullStand = resolveInt(values, "pull_stand", row.getPullStand(), errors);
        Integer indepStand3 = resolveInt(values, "indep_stand_3", row.getIndepStand3(), errors);
        Integer indepStand20 = resolveInt(values, "indep_stand_20", row.getIndepStand20(), errors);
        Integer oneLegHandR = resolveInt(values, "one_leg_hand_r", row.getOneLegHandR(), errors);
        Integer oneLegHandL = resolveInt(values, "one_leg_hand_l", row.getOneLegHandL(), errors);
        Integer oneLegR = resolveInt(values, "one_leg_r", row.getOneLegR(), errors);
        Integer oneLegL = resolveInt(values, "one_leg_l", row.getOneLegL(), errors);
        Integer shortSitStand = resolveInt(values, "short_sit_stand", row.getShortSitStand(), errors);
        Integer highKneeStandR = resolveInt(values, "high_knee_stand_r", row.getHighKneeStandR(), errors);
        Integer highKneeStandL = resolveInt(values, "high_knee_stand_l", row.getHighKneeStandL(), errors);
        Integer lowerFloor = resolveInt(values, "lower_floor", row.getLowerFloor(), errors);
        Integer squats = resolveInt(values, "squats", row.getSquats(), errors);
        Integer standPickPen = resolveInt(values, "stand_pick_pen", row.getStandPickPen(), errors);
        Integer gmfmModA = resolveInt(values, "gmfm_mod_a", row.getGmfmModA(), errors);
        Integer gmfmModB = resolveInt(values, "gmfm_mod_b", row.getGmfmModB(), errors);
        Integer gmfmModC = resolveInt(values, "gmfm_mod_c", row.getGmfmModC(), errors);
        Integer gmfmModE = resolveInt(values, "gmfm_mod_e", row.getGmfmModE(), errors);

        if (!errors.isEmpty()) {
            throw new PtEvaluationValidationException("Some general fields are invalid.", errors);
        }

        general.updateVisitMeasurements(visitId, height, weight, rFootLength, lFootLength, rFootWidth, lFootWidth);
        general.updateDataPt(row.getDataPtId(), ambScale, gmfcs, fms5, fms50, fms500, comments, pullStand,
                indepStand3, indepStand20, oneLegHandR, oneLegHandL, oneLegR, oneLegL, shortSitStand,
                highKneeStandR, highKneeStandL, lowerFloor, squats, standPickPen, gmfmModA, gmfmModB, gmfmModC,
                gmfmModE);

        return getGeneral(patientId, visitId);
    }

    private static String resolveText(Map<String, Object> values, String key, String currentValue) {
        if (!values.containsKey(key)) {
            return currentValue;
        }
        Object raw = values.get(key);
        String submitted = raw == null ? null : raw.toString();
        return (submitted == null || submitted.isEmpty()) ? null : submitted;
    }

    private static Float resolveFloat(Map<String, Object> values, String key, Float currentValue,
            Map<String, String> errors) {
        if (!values.containsKey(key)) {
            return currentValue;
        }
        Object raw = values.get(key);
        String submitted = raw == null ? null : raw.toString();
        if (submitted == null || submitted.isEmpty()) {
            return null;
        }
        try {
            return Float.parseFloat(submitted);
        } catch (NumberFormatException e) {
            errors.put(key, "Must be a number.");
            return currentValue;
        }
    }

    private static Integer resolveInt(Map<String, Object> values, String key, Integer currentValue,
            Map<String, String> errors) {
        if (!values.containsKey(key)) {
            return currentValue;
        }
        Object raw = values.get(key);
        String submitted = raw == null ? null : raw.toString();
        if (submitted == null || submitted.isEmpty()) {
            return null;
        }
        try {
            return Integer.parseInt(submitted);
        } catch (NumberFormatException e) {
            errors.put(key, "Must be a whole number.");
            return currentValue;
        }
    }

    private static Integer resolveFromSet(Map<String, Object> values, String key, Integer currentValue,
            java.util.Set<Integer> allowed, Map<String, String> errors) {
        Integer resolved = resolveInt(values, key, currentValue, errors);
        if (resolved != null && !allowed.contains(resolved)) {
            errors.put(key, "Invalid value.");
            return currentValue;
        }
        return resolved;
    }

    private interface IdExists {
        boolean test(int id);
    }

    private static Integer resolveLookupId(Map<String, Object> values, String key, Integer currentValue,
            IdExists exists, Map<String, String> errors) {
        Integer resolved = resolveInt(values, key, currentValue, errors);
        if (resolved != null && !exists.test(resolved)) {
            errors.put(key, "Unknown value.");
            return currentValue;
        }
        return resolved;
    }

    public GeneralEvaluation createRecord(int patientId, int visitId) {
        if (visits.findPtGeneralEvaluationByVisitIdAndPatientId(visitId, patientId).isPresent()) {
            throw new ResponseStatusException(HttpStatus.CONFLICT, "General data already exists for this visit.");
        }
        general.insertBlankDataPt(visitId);
        return getGeneral(patientId, visitId);
    }
}














package org.nemours.gaitlab.service;

import java.util.List;

/**
 * Shared response shape for the five read-only PT evaluation tabs ported from
 * lab/data/pt-eval-{prom,orth,mc,tone,pain}.jsp (Gait / non-movement-disorder visits only).
 * Each tab's service builds the sections; a generic frontend renderer displays them.
 */
public record PtEvaluationTab(
        Integer patientId, Integer visitId, String tab,
        boolean hasData, String emptyMessage,
        List<Section> sections, PtEvaluationActions actions) {

    /** Returns a copy of this response with the given actions attached - lets the tab-dispatch
     * layer own role/movement-disorder gating without the individual tab services (which don't
     * know about the current user) needing to. */
    public PtEvaluationTab withActions(PtEvaluationActions actions) {
        return new PtEvaluationTab(patientId, visitId, tab, hasData, emptyMessage, sections, actions);
    }

    public record ColumnGroup(String label, int span) {
    }

    /** Per-row actions, currently only meaningful for Orthotics rows (edit/delete the trial). */
    public record RowActions(boolean edit, boolean delete) {
    }

    public record Row(String label, List<String> cells, List<Integer> shadedCells, Integer id, RowActions actions) {
        public Row(String label, List<String> cells) {
            this(label, cells, List.of(), null, null);
        }
    }

    public record Section(
            String kind, String title,
            List<ColumnGroup> columnGroups, List<String> columns, List<Row> rows,
            List<String> items, String text, String footnote) {

        public static Section table(String title, List<ColumnGroup> columnGroups, List<String> columns,
                List<Row> rows, String footnote) {
            return new Section("table", title, columnGroups, columns, rows, null, null, footnote);
        }

        public static Section list(String title, List<String> items) {
            return new Section("list", title, null, null, null, items, null, null);
        }

        public static Section text(String title, String text) {
            return new Section("text", title, null, null, null, null, text, null);
        }
    }

    public static PtEvaluationTab empty(Integer patientId, Integer visitId, String tab, String emptyMessage) {
        return new PtEvaluationTab(patientId, visitId, tab, false, emptyMessage, List.of(), null);
    }

    public static PtEvaluationTab of(Integer patientId, Integer visitId, String tab, List<Section> sections) {
        return new PtEvaluationTab(patientId, visitId, tab, true, null, sections, null);
    }
}










package org.nemours.gaitlab.service;

import java.util.List;

/** GET .../pt-evaluation/tabs - lets the frontend build its tab nav from the backend rather than
 * guessing visit type client-side. Order and labels match pt-tab-header.jsp exactly for both
 * variants. */
public record PtEvaluationTabList(boolean movementDisorder, List<TabEntry> tabs) {

    public record TabEntry(String key, String label) {
    }
}














package org.nemours.gaitlab.service;

import java.util.List;

import org.nemours.gaitlab.repository.VisitRepository;
import org.nemours.gaitlab.repository.VisitRepository.InterpretationVisit;
import org.nemours.gaitlab.requests.SavePtEvaluationFormRequest;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.web.server.ResponseStatusException;

/** Dispatches GET/PUT/POST /api/patients/{patientId}/visits/{visitId}/pt-evaluation/{tab}(/form|
 * /record) to the right tab service, for both Gait and movement-disorder visits. Phase 2: only
 * Motor Control and Tone have form/save/create-record wired up so far for Gait - the other cases
 * 404 until their turn. */
@Service
public class PtEvaluationTabsService {
    private final VisitRepository visits;
    private final PromStrengthService promStrength;
    private final OrthosesService orthoses;
    private final MotorControlToneService motorControlTone;
    private final PainService pain;
    private final HatService hat;
    private final DystoniaService dystonia;

    public PtEvaluationTabsService(VisitRepository visits, PromStrengthService promStrength,
            OrthosesService orthoses, MotorControlToneService motorControlTone, PainService pain, HatService hat,
            DystoniaService dystonia) {
        this.visits = visits;
        this.promStrength = promStrength;
        this.orthoses = orthoses;
        this.motorControlTone = motorControlTone;
        this.pain = pain;
        this.hat = hat;
        this.dystonia = dystonia;
    }

    /** GET .../pt-evaluation/tabs - order and labels match pt-tab-header.jsp exactly. */
    public PtEvaluationTabList getTabs(int patientId, int visitId) {
        boolean movementDisorder = isMovementDisorder(patientId, visitId);
        List<PtEvaluationTabList.TabEntry> tabs = movementDisorder
                ? List.of(
                        new PtEvaluationTabList.TabEntry("general-md", "General"),
                        new PtEvaluationTabList.TabEntry("prom-strength", "PROM/Strength"),
                        new PtEvaluationTabList.TabEntry("orthotics", "Orthotics"),
                        new PtEvaluationTabList.TabEntry("tone", "Tone"),
                        new PtEvaluationTabList.TabEntry("pain", "Pain"),
                        new PtEvaluationTabList.TabEntry("hat", "HAT"),
                        new PtEvaluationTabList.TabEntry("dystonia-scale", "Dystonia Scale"))
                : List.of(
                        new PtEvaluationTabList.TabEntry("general", "General & GMFM"),
                        new PtEvaluationTabList.TabEntry("prom-strength", "PROM/Strength"),
                        new PtEvaluationTabList.TabEntry("orthotics", "Orthotics"),
                        new PtEvaluationTabList.TabEntry("motor-control", "Motor Control"),
                        new PtEvaluationTabList.TabEntry("tone", "Tone"),
                        new PtEvaluationTabList.TabEntry("pain", "Pain"));
        return new PtEvaluationTabList(movementDisorder, tabs);
    }

    private boolean isMovementDisorder(int patientId, int visitId) {
        Integer visitTypeId = requireVisit(patientId, visitId).getVisitTypeId();
        return PtEvaluationActions.isMovementDisorder(visitTypeId);
    }

    public PtEvaluationTab get(int patientId, int visitId, String tab, boolean isAdmin) {
        boolean movementDisorder = isMovementDisorder(patientId, visitId);

        PtEvaluationTab result = switch (tab) {
            case "prom-strength" -> promStrength.build(patientId, visitId, tab, movementDisorder);
            case "orthotics" -> orthoses.build(patientId, visitId, tab, isAdmin, movementDisorder);
            case "motor-control" -> motorControlTone.buildMotorControl(patientId, visitId, tab);
            case "tone" -> motorControlTone.buildTone(patientId, visitId, tab, movementDisorder);
            case "pain" -> pain.build(patientId, visitId, tab);
            case "hat" -> hat.build(patientId, visitId, tab);
            case "dystonia-scale" -> dystonia.build(patientId, visitId, tab);
            default -> throw new ResponseStatusException(HttpStatus.NOT_FOUND);
        };
        return result.withActions(actionsFor(tab, isAdmin, movementDisorder, patientId, visitId));
    }

    public PtEvaluationForm getForm(int patientId, int visitId, String tab) {
        boolean movementDisorder = isMovementDisorder(patientId, visitId);
        return switch (tab) {
            case "motor-control" -> motorControlTone.buildMotorControlForm(visitId);
            case "tone" -> motorControlTone.buildToneForm(visitId, movementDisorder);
            case "pain" -> pain.buildForm(patientId, visitId);
            case "prom-strength" -> promStrength.buildForm(visitId, movementDisorder);
            case "hat" -> hat.buildForm(visitId);
            case "dystonia-scale" -> dystonia.buildForm(visitId);
            case "orthotics" -> movementDisorder ? orthoses.buildCommentsForm(patientId, visitId)
                    : throwNotFound();
            default -> throw new ResponseStatusException(HttpStatus.NOT_FOUND);
        };
    }

    private static <T> T throwNotFound() {
        throw new ResponseStatusException(HttpStatus.NOT_FOUND);
    }

    /** edit is admin-only for PROM/Strength only, same as the view's actions.edit rule -
     * enforced here too since legacy's own server-side insert/update code has no such check at
     * all, this is purely this port's added guardrail. Movement-disorder Orthotics Comments is
     * deliberately NOT admin-gated here either, matching pt-eval-orth.jsp's own textarea/Save. */
    public PtEvaluationTab saveForm(int patientId, int visitId, String tab, boolean isAdmin,
            SavePtEvaluationFormRequest request) {
        boolean movementDisorder = isMovementDisorder(patientId, visitId);
        if (tab.equals("prom-strength") && !isAdmin) {
            throw new ResponseStatusException(HttpStatus.FORBIDDEN);
        }
        PtEvaluationTab result = switch (tab) {
            case "motor-control" ->
                motorControlTone.saveMotorControl(patientId, visitId, request.revision(), request.values());
            case "tone" -> motorControlTone.saveTone(patientId, visitId, request.revision(), request.values(),
                    movementDisorder);
            case "pain" -> pain.save(patientId, visitId, request.revision(), request.values());
            case "prom-strength" -> promStrength.save(patientId, visitId, request.revision(), request.values(),
                    movementDisorder);
            case "hat" -> hat.save(patientId, visitId, request.revision(), request.values());
            case "dystonia-scale" -> dystonia.save(patientId, visitId, request.revision(), request.values());
            case "orthotics" -> movementDisorder
                    ? orthoses.saveComments(patientId, visitId, request.revision(), request.values())
                    : throwNotFound();
            default -> throw new ResponseStatusException(HttpStatus.NOT_FOUND);
        };
        return result.withActions(actionsFor(tab, isAdmin, movementDisorder, patientId, visitId));
    }

    public PtEvaluationTab createRecord(int patientId, int visitId, String tab, boolean isAdmin) {
        boolean movementDisorder = isMovementDisorder(patientId, visitId);
        if (!isAdmin) {
            throw new ResponseStatusException(HttpStatus.FORBIDDEN);
        }
        PtEvaluationTab result = switch (tab) {
            case "motor-control" -> motorControlTone.createMotorControlRecord(patientId, visitId);
            case "tone" -> motorControlTone.createToneRecord(patientId, visitId, movementDisorder);
            case "pain" -> pain.createRecord(patientId, visitId);
            case "prom-strength" -> promStrength.createRecord(patientId, visitId, movementDisorder);
            case "hat" -> hat.createRecord(patientId, visitId);
            case "dystonia-scale" -> dystonia.createRecord(patientId, visitId);
            case "orthotics" -> movementDisorder ? orthoses.createCommentsRecord(patientId, visitId)
                    : throwNotFound();
            default -> throw new ResponseStatusException(HttpStatus.NOT_FOUND);
        };
        return result.withActions(actionsFor(tab, isAdmin, movementDisorder, patientId, visitId));
    }

    private InterpretationVisit requireVisit(int patientId, int visitId) {
        return visits.findInterpretationVisitByIdAndPatientId(visitId, patientId)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));
    }

    /**
     * create (Create New Record) is admin-only on every tab. edit is admin-only only for
     * PROM/Strength's "Edit Data" - Motor Control/Tone/Pain values are editable by any
     * authenticated user once a record exists. add (Add an Orthosis) is admin-only. report (the
     * Comparison Report button) is always false here - pt.jsp (General) is the only legacy page
     * with that button; none of these tab pages have it.
     *
     * <p>Orthotics on a movement-disorder visit is a special case: create is admin-only AND only
     * when no data_pt_movement_disorder row exists yet; edit (the Comments textarea/Save) has no
     * admin gate and is only true once that row exists - pt-eval-orth.jsp:206-228. Gait Orthotics
     * keeps its own create:false/edit:false (Add an Orthosis is a separate add-only action).
     */
    private PtEvaluationActions actionsFor(String tab, boolean isAdmin, boolean movementDisorder, int patientId,
            int visitId) {
        if (tab.equals("orthotics")) {
            if (!movementDisorder) {
                return new PtEvaluationActions(false, false, isAdmin, false);
            }
            return new PtEvaluationActions(orthoses.commentsCreatable(patientId, visitId, isAdmin),
                    orthoses.commentsEditable(patientId, visitId), isAdmin, false);
        }
        return switch (tab) {
            case "prom-strength" -> new PtEvaluationActions(isAdmin, isAdmin, false, false);
            default -> new PtEvaluationActions(isAdmin, true, false, false);
        };
    }
}














package org.nemours.gaitlab.service;

import java.util.Map;

/** Carries field-level validation failures for the PT Evaluation write endpoints, rendered as
 * 400 { "message", "fieldErrors": { name: msg } } by the owning controller. */
public class PtEvaluationValidationException extends RuntimeException {
    private final Map<String, String> fieldErrors;

    public PtEvaluationValidationException(String message, Map<String, String> fieldErrors) {
        super(message);
        this.fieldErrors = fieldErrors;
    }

    public Map<String, String> fieldErrors() {
        return fieldErrors;
    }

    public record Response(String message, Map<String, String> fieldErrors) {
    }
}
















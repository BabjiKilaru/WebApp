package org.nemours.gaitlab.controller;
import org.nemours.gaitlab.model.Age;
import org.nemours.gaitlab.model.BodyLocation;
import org.nemours.gaitlab.model.Condition;
import org.nemours.gaitlab.model.HistoryCondition;
import org.nemours.gaitlab.model.Surgery;
import org.nemours.gaitlab.model.SeizureMedication;
import org.nemours.gaitlab.repository.AgeRepository;
import org.nemours.gaitlab.repository.BodyLocationRepository;
import org.nemours.gaitlab.repository.ConditionRepository;
import org.nemours.gaitlab.repository.FrequencyRepository;
import org.nemours.gaitlab.repository.HistoryConditionRepository;
import org.nemours.gaitlab.repository.SeizureMedicationRepository;
import org.nemours.gaitlab.repository.SurgeryRepository;
import org.nemours.gaitlab.repository.UeHistoryQuestionnaireRepository;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api/reference")
public class ReferenceController {
    private final SurgeryRepository surgeryRepository;
    private final BodyLocationRepository bodyLocationRepository;
    private final AgeRepository ageRepository;
    private final ConditionRepository conditionRepository;
    private final FrequencyRepository frequencyRepository;
    private final HistoryConditionRepository historyConditionRepository;
    private final SeizureMedicationRepository seizureMedicationRepository;
    private final UeHistoryQuestionnaireRepository ueHistoryQuestionnaireRepository;

    public ReferenceController(SurgeryRepository surgeryRepository, BodyLocationRepository bodyLocationRepository, AgeRepository ageRepository, ConditionRepository conditionRepository, FrequencyRepository frequencyRepository, HistoryConditionRepository historyConditionRepository, SeizureMedicationRepository seizureMedicationRepository, UeHistoryQuestionnaireRepository ueHistoryQuestionnaireRepository) {
        this.surgeryRepository = surgeryRepository;
        this.bodyLocationRepository = bodyLocationRepository;
        this.ageRepository = ageRepository;
        this.conditionRepository = conditionRepository;
        this.frequencyRepository = frequencyRepository;
        this.historyConditionRepository = historyConditionRepository;
        this.seizureMedicationRepository = seizureMedicationRepository;
        this.ueHistoryQuestionnaireRepository = ueHistoryQuestionnaireRepository;
    }

    public record SideOption(String value, String label) {}

    @GetMapping("/surgeries")
    public List<Surgery> getSurgeries() {
        return surgeryRepository.findAllByOrderByNameAsc();
    }

    @GetMapping("surgery-sides")
    public List<SideOption> getSurgerySides() {
        return List.of(
            new SideOption("L", "Left"),
            new SideOption("R", "Right"),
            new SideOption("Both", "Both"),
            new SideOption("NA", "N/A")
        );
    }

    @GetMapping("/body-locations")
    public Object getBodyLocations(@RequestParam(required = false) String language) {
        if (language == null) {
            return bodyLocationRepository.findAllByOrderByLocationAsc().stream().map(BodyLocation::getLocation).toList();
        }
        return bodyLocationRepository.findAllWithSpanish().stream().map(row -> localized((String) row[0], (String) row[1], language)).toList();
    }

    @GetMapping("/botox-sides")
    public List<SideOption> getBotoxSides() {
        return List.of(
            new SideOption("L", "Left"),
            new SideOption("R", "Right"),
            new SideOption("Both", "Both")
        );
    }

    @GetMapping("/history-conditions")
    public Object getHistoryConditions(@RequestParam(required = false) String language) {
        if (language == null) {
            return historyConditionRepository.findAllByOrderByConditionAsc();
        }
        return historyConditionRepository.findAllWithSpanish().stream().map(row -> Map.of("code", (String) row[0], "name", labelOf((String) row[1], (String) row[2], language))).toList();
    }

    @GetMapping("/ages")
    public Object getAges(@RequestParam(required = false) String language) {
        if (language == null) {
            return ageRepository.findAllByOrderByChronAsc().stream().map(Age::getAge).toList();
        }
        return ageRepository.findAllWithSpanish().stream().map(row -> localized((String) row[0], (String) row[1], language)).toList();
    }

    @GetMapping("/conditions")
    public Object getConditions(@RequestParam(required = false) String language) {
        if (language == null) {
            return conditionRepository.findAllByOrderByConditionAsc();
        }
        return conditionRepository.findAllWithSpanish().stream().map(row -> Map.of("code", (String) row[0], "name", labelOf((String) row[1], (String) row[2], language))).toList();
    }

    @GetMapping("/frequencies")
    public List<Map<String, Object>> getFrequencies(@RequestParam(required = false) String language) {
        return frequencyRepository.findAllOrderByCodeDesc().stream().map(row -> {
            Map<String, Object> result = new LinkedHashMap<>();
            result.put("value", String.valueOf(row.get("code")));
            result.put("label", labelOf((String) row.get("freq"), (String) row.get("il8n_sp"), language));
            return result;
        })
        .toList();
    }

    @GetMapping("/seizure-meds")
    public Object getSeizureMedications(@RequestParam(required = false) String language) {
        List<SeizureMedication> medications = seizureMedicationRepository.findAllByOrderByMedAsc().stream().filter(medication -> medication.getMed() != null && !medication.getMed().isBlank()).toList();
        if (language == null) {
            return medications.stream().map(SeizureMedication::getMed).toList();
        }
        return medications.stream().map(medication -> Map.of("value", medication.getMed(), "label", medication.getMed())).toList();
    }

    @GetMapping("/ue-history-questions")
    public Map<String, Object> getUeHistoryQuestions() {
        Map<String, Object> result = new LinkedHashMap<>();
        result.put("handlesObjects", ueHistoryQuestionnaireRepository.findQuestionBundle(54));
        result.put("dresses", ueHistoryQuestionnaireRepository.findQuestionBundle(55));
        result.put("bathes", ueHistoryQuestionnaireRepository.findQuestionBundle(56));
        result.put("toilets", ueHistoryQuestionnaireRepository.findQuestionBundle(57));
        result.put("armHandConcerns", ueHistoryQuestionnaireRepository.findQuestionBundle(58));
        return result;
    }

    private String labelOf(String english, String spanish, String language) {
        if ("sp".equalsIgnoreCase(language) && spanish != null && !spanish.isBlank()) {
            return spanish;
        }
        return english;
    }

    private Map<String, Object> localized(String value, String spanish, String language) {
        Map<String, Object> result = new LinkedHashMap<>();
        result.put("value", value);
        result.put("label", labelOf(value, spanish, language));
        return result;
    }
}










package org.nemours.gaitlab.repository;

import org.nemours.gaitlab.model.Age;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;

import java.util.List;

public interface AgeRepository extends JpaRepository<Age, String> {
    List<Age> findAllByOrderByChronAsc();

    @Query(value = "SELECT age, age FROM public.ages2 ORDER BY chron", nativeQuery = true)
    List<Object[]> findAllWithSpanish();
}











package org.nemours.gaitlab.repository;

import org.nemours.gaitlab.model.BodyLocation;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;

import java.util.List;

public interface BodyLocationRepository extends JpaRepository<BodyLocation, String> {
    List<BodyLocation> findAllByOrderByLocationAsc();

    @Query(value = "SELECT location, il8n_sp FROM public.body_loc ORDER BY location", nativeQuery = true)
    List<Object[]> findAllWithSpanish();
}








package org.nemours.gaitlab.repository;

import org.nemours.gaitlab.model.Condition;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;

import java.util.List;

public interface ConditionRepository extends JpaRepository<Condition, String> {
    List<Condition> findAllByOrderByConditionAsc();

    @Query(value = "SELECT code, condition, il8n_sp FROM public.conditions ORDER BY condition", nativeQuery = true)
    List<Object[]> findAllWithSpanish();
}









package org.nemours.gaitlab.repository;

import org.nemours.gaitlab.model.HistoryCondition;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;

import java.util.List;

public interface HistoryConditionRepository extends JpaRepository<HistoryCondition, String> {
    List<HistoryCondition> findAllByOrderByConditionAsc();

    @Query(value = "SELECT code, condition, il8n_sp FROM public.history_cond ORDER BY condition", nativeQuery = true)
    List<Object[]> findAllWithSpanish();
}









package org.nemours.gaitlab.repository;

import java.util.List;
import java.time.LocalDate;

import org.nemours.gaitlab.model.Visit;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

@Repository
public interface VisitRepository extends JpaRepository<Visit, Integer> {

    @Query(value = """
            SELECT v.id AS "id", v.visit_type_id AS "visitTypeId"
            FROM public.visits v
            WHERE v.id = :visitId AND v.pt_id = :patientId
            """, nativeQuery = true)
    java.util.Optional<InterpretationVisit> findInterpretationVisitByIdAndPatientId(
            @Param("visitId") Integer visitId,
            @Param("patientId") Integer patientId);

    interface InterpretationVisit {
        Integer getVisitTypeId();
    }

    List<Visit> findByPatientId(Integer patientId);

    List<Visit> findByPatientIdOrderByDateDesc(Integer patientId);

    @Query(value = """
            SELECT v.id AS id, v.date::text AS "visitDate"
            FROM public.visits v
            WHERE v.pt_id = :patientId
            ORDER BY v.date DESC, v.id DESC
            """, nativeQuery = true)
    List<VideoVisitDate> findVideoVisitDatesByPatientId(@Param("patientId") Integer patientId);

    @Query(value = """
            SELECT v.id AS id, v.date::text AS "visitDate"
            FROM public.visits v
            WHERE v.pt_id = :patientId AND v.id = :visitId
            """, nativeQuery = true)
    java.util.Optional<VideoVisitDate> findVideoVisitDateByIdAndPatientId(
            @Param("visitId") Integer visitId,
            @Param("patientId") Integer patientId);

    List<Visit> findByPatientIdAndDateOrderByIdAsc(
        Integer patientId,
        LocalDate date
    );

    @Query(
        value = """
                SELECT COALESCE(MAX(id), 0) + 1
                FROM public.visits
                """,
            nativeQuery = true
    )
    Integer getNextVisitId();

    @Query(value = """
            SELECT 
	            v.id, 
	            v.pt_id, 
                v.date,
                v.attendance,
	            CASE WHEN vt.name IS NULL THEN v.visit_type ELSE vt.name END AS visit_type,
	            tg.name AS visitSubType
            FROM public.visits v
            LEFT JOIN public.new_visit_type vt ON v.visit_type_id = vt.id
            LEFT JOIN public.new_test_group tg ON v.test_group_id = tg.id
            WHERE v.pt_id = :patientId
            ORDER BY v.date DESC, v.id DESC;
            """, nativeQuery = true)
    List<Object[]> findVisitDisplayByPatientId( @Param("patientId") Integer patientId);

    @Query(value = """
            SELECT 
                v.id AS visit_id,
                p.id AS patient_id,
                p.fname,
                p.lname,
                p.mrn,
                CASE 
                    WHEN vt.name IS NULL
                    THEN v.visit_type
                    ELSE vt.name
                END AS visit_type,
                tg.name AS visit_sub_type
            FROM public.patients p
            JOIN public.visits v
                ON v.pt_id = p.id
            LEFT JOIN public.new_visit_type vt
                ON v.visit_type_id = vt.id
            LEFT JOIN public.new_test_group tg
                ON v.test_group_id = tg.id
            WHERE v.date = :date
                AND (v.attendance IS NULL
                    OR v.attendance = ''
                    OR v.attendance = 'Yes')
            ORDER BY p.lname, p.fname, v.id
            """,
            nativeQuery = true
        )
    List<Object[]> findScheduledVisitsByDate(@Param("date") LocalDate date);

    interface VideoVisitDate {
        Integer getId();

        String getVisitDate();
    }

    @Query(value = """
            SELECT
                dp.id AS "dataPtId",
                v.date AS "visitDate",
                v.icd9 AS "icd9",
                p.dob AS "dob",
                p.gender AS "gender",
                p.gen_diag AS "genDiag",
                v.height AS "heightCm",
                v.weight AS "weightKg",
                v.r_foot_length AS "rightFootLengthCm",
                v.l_foot_length AS "leftFootLengthCm",
                v.r_foot_width AS "rightFootWidthCm",
                v.l_foot_width AS "leftFootWidthCm",
                dp.amb_scale AS "ambScaleId",
                amb.description AS "ambulationScale",
                dp.gmfcs AS "gmfcsId",
                dp.fms_5 AS "fms5Id",
                fms5.label AS "fms5m",
                fms5.description AS "fms5mDesc",
                dp.fms_50 AS "fms50Id",
                fms50.label AS "fms50m",
                fms50.description AS "fms50mDesc",
                dp.fms_500 AS "fms500Id",
                fms500.label AS "fms500m",
                fms500.description AS "fms500mDesc",
                dp.comments AS "comments",
                dp.pull_stand AS "pullStand",
                dp.indep_stand_3 AS "indepStand3",
                dp.indep_stand_20 AS "indepStand20",
                dp.one_leg_hand_r AS "oneLegHandR",
                dp.one_leg_hand_l AS "oneLegHandL",
                dp.one_leg_r AS "oneLegR",
                dp.one_leg_l AS "oneLegL",
                dp.short_sit_stand AS "shortSitStand",
                dp.high_knee_stand_r AS "highKneeStandR",
                dp.high_knee_stand_l AS "highKneeStandL",
                dp.lower_floor AS "lowerFloor",
                dp.squats AS "squats",
                dp.stand_pick_pen AS "standPickPen",
                dp.gmfm_mod_a AS "gmfmModA",
                dp.gmfm_mod_b AS "gmfmModB",
                dp.gmfm_mod_c AS "gmfmModC",
                dp.gmfm_mod_e AS "gmfmModE"
            FROM public.visits v
            JOIN public.data_pt dp ON dp.visit_id = v.id
            JOIN public.patients p ON p.id = v.pt_id
            LEFT JOIN public.amb_scale amb ON amb.id = dp.amb_scale
            LEFT JOIN public.fms_list fms5 ON fms5.id = dp.fms_5
            LEFT JOIN public.fms_list fms50 ON fms50.id = dp.fms_50
            LEFT JOIN public.fms_list fms500 ON fms500.id = dp.fms_500
            WHERE v.id = :visitId AND v.pt_id = :patientId
            """, nativeQuery = true)
    java.util.Optional<PtGeneralEvaluationRow> findPtGeneralEvaluationByVisitIdAndPatientId(
            @Param("visitId") Integer visitId,
            @Param("patientId") Integer patientId);

    interface PtGeneralEvaluationRow {
        Integer getDataPtId();

        LocalDate getVisitDate();

        Integer getIcd9();

        LocalDate getDob();

        String getGender();

        Integer getGenDiag();

        Float getHeightCm();

        Float getWeightKg();

        Float getRightFootLengthCm();

        Float getLeftFootLengthCm();

        Float getRightFootWidthCm();

        Float getLeftFootWidthCm();

        Integer getAmbScaleId();

        String getAmbulationScale();

        Integer getGmfcsId();

        Integer getFms5Id();

        String getFms5m();

        String getFms5mDesc();

        Integer getFms50Id();

        String getFms50m();

        String getFms50mDesc();

        Integer getFms500Id();

        String getFms500m();

        String getFms500mDesc();

        String getComments();

        Integer getPullStand();

        Integer getIndepStand3();

        Integer getIndepStand20();

        Integer getOneLegHandR();

        Integer getOneLegHandL();

        Integer getOneLegR();

        Integer getOneLegL();

        Integer getShortSitStand();

        Integer getHighKneeStandR();

        Integer getHighKneeStandL();

        Integer getLowerFloor();

        Integer getSquats();

        Integer getStandPickPen();

        Integer getGmfmModA();

        Integer getGmfmModB();

        Integer getGmfmModC();

        Integer getGmfmModE();
    }

    @Query(value = "SELECT id AS \"id\", label AS \"label\", description AS \"description\" FROM public.gmfcs_list",
            nativeQuery = true)
    List<GmfcsListEntry> findAllGmfcsListEntries();

    interface GmfcsListEntry {
        Integer getId();

        String getLabel();

        String getDescription();
    }

    /** Movement-disorder counterpart of findPtGeneralEvaluationByVisitIdAndPatientId - pt.jsp's
     * isMovementDisorder branch (mainTable=data_pt_movement_disorder), joined with macs_list/
     * cfcs_list the same way pt.jsp's own rsPtAge query does. No GMFM-D task columns - that table
     * has none; Other GMFM Modules and the GMFCS calculation are Gait-only. */
    @Query(value = """
            SELECT
                dp.id AS "dataPtMdId",
                v.date AS "visitDate",
                v.icd9 AS "icd9",
                p.dob AS "dob",
                p.gender AS "gender",
                p.gen_diag AS "genDiag",
                v.height AS "heightCm",
                v.weight AS "weightKg",
                v.r_foot_length AS "rightFootLengthCm",
                v.l_foot_length AS "leftFootLengthCm",
                v.r_foot_width AS "rightFootWidthCm",
                v.l_foot_width AS "leftFootWidthCm",
                dp.amb_scale AS "ambScaleId",
                amb.description AS "ambulationScale",
                dp.gmfcs AS "gmfcsId",
                gmfcs.label AS "gmfcsLabel",
                gmfcs.description AS "gmfcsDesc",
                dp.fms_5 AS "fms5Id",
                fms5.label AS "fms5m",
                fms5.description AS "fms5mDesc",
                dp.fms_50 AS "fms50Id",
                fms50.label AS "fms50m",
                fms50.description AS "fms50mDesc",
                dp.fms_500 AS "fms500Id",
                fms500.label AS "fms500m",
                fms500.description AS "fms500mDesc",
                dp.macs AS "macsId",
                macs.label AS "macsLabel",
                macs.description AS "macsDesc",
                dp.cfcs AS "cfcsId",
                cfcs.label AS "cfcsLabel",
                cfcs.description AS "cfcsDesc",
                dp.comments AS "comments",
                dp.orthoses AS "orthoses"
            FROM public.visits v
            JOIN public.data_pt_movement_disorder dp ON dp.visit_id = v.id
            JOIN public.patients p ON p.id = v.pt_id
            LEFT JOIN public.amb_scale amb ON amb.id = dp.amb_scale
            LEFT JOIN public.gmfcs_list gmfcs ON gmfcs.id = dp.gmfcs
            LEFT JOIN public.fms_list fms5 ON fms5.id = dp.fms_5
            LEFT JOIN public.fms_list fms50 ON fms50.id = dp.fms_50
            LEFT JOIN public.fms_list fms500 ON fms500.id = dp.fms_500
            LEFT JOIN public.macs_list macs ON macs.id = dp.macs
            LEFT JOIN public.cfcs_list cfcs ON cfcs.id = dp.cfcs
            WHERE v.id = :visitId AND v.pt_id = :patientId
            """, nativeQuery = true)
    java.util.Optional<PtGeneralMdEvaluationRow> findPtGeneralMdEvaluationByVisitIdAndPatientId(
            @Param("visitId") Integer visitId,
            @Param("patientId") Integer patientId);

    interface PtGeneralMdEvaluationRow {
        Integer getDataPtMdId();

        LocalDate getVisitDate();

        Integer getIcd9();

        LocalDate getDob();

        String getGender();

        Integer getGenDiag();

        Float getHeightCm();

        Float getWeightKg();

        Float getRightFootLengthCm();

        Float getLeftFootLengthCm();

        Float getRightFootWidthCm();

        Float getLeftFootWidthCm();

        Integer getAmbScaleId();

        String getAmbulationScale();

        Integer getGmfcsId();

        String getGmfcsLabel();

        String getGmfcsDesc();

        Integer getFms5Id();

        String getFms5m();

        String getFms5mDesc();

        Integer getFms50Id();

        String getFms50m();

        String getFms50mDesc();

        Integer getFms500Id();

        String getFms500m();

        String getFms500mDesc();

        Integer getMacsId();

        String getMacsLabel();

        String getMacsDesc();

        Integer getCfcsId();

        String getCfcsLabel();

        String getCfcsDesc();

        String getComments();

        /** pt-eval-orth.jsp's own MD-only "Comments" block - data_pt_movement_disorder.orthoses,
         * nothing to do with the orthoses table/rows above it on the same page. */
        String getOrthoses();
    }

    /** Patient/visit identity fields for the Comparison Report header - eval-report_combined_new
     * .jsp's own pt_info/visit_info queries, the parts that aren't already covered by
     * findPtGeneralEvaluationByVisitIdAndPatientId (name/MRN/therapist/interpreting doctor). */
    @Query(value = """
            SELECT
                p.fname AS "firstName", p.lname AS "lastName", p.mrn AS "mrn",
                p.dob AS "dob", p.gender AS "gender", v.date AS "visitDate",
                therapist.fname AS "therapistFirstName", therapist.lname AS "therapistLastName",
                interpreter.fname AS "interpretingDoctorFirstName",
                interpreter.lname AS "interpretingDoctorLastName"
            FROM public.visits v
            JOIN public.patients p ON p.id = v.pt_id
            LEFT JOIN public.users therapist ON therapist.id = v.visit_pt
            LEFT JOIN public.users interpreter ON interpreter.id = v.interp_md
            WHERE v.id = :visitId AND v.pt_id = :patientId
            """, nativeQuery = true)
    java.util.Optional<ComparisonHeaderRow> findComparisonHeaderByVisitIdAndPatientId(
            @Param("visitId") Integer visitId,
            @Param("patientId") Integer patientId);

    interface ComparisonHeaderRow {
        String getFirstName();

        String getLastName();

        String getMrn();

        LocalDate getDob();

        String getGender();

        LocalDate getVisitDate();

        String getTherapistFirstName();

        String getTherapistLastName();

        String getInterpretingDoctorFirstName();

        String getInterpretingDoctorLastName();
    }
}












package org.nemours.gaitlab.service;

import java.util.ArrayList;
import java.util.List;
import org.nemours.gaitlab.repository.InterpretationRepository;
import org.nemours.gaitlab.repository.InterpretationRepository.Measurements;
import org.nemours.gaitlab.repository.VisitRepository;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.web.server.ResponseStatusException;

@Service
public class InterpretationService {
    public enum VisitKind {
        // 11 matches new_visit_type.id for "Movement Disorder" and legacy's own
        // VisitType.MOVEMENT_DISORDER_ID - the id 9 used here before didn't exist as a
        // new_visit_type row at all, so this case could never match (found during Phase 4 PT
        // Evaluation live verification: visit_type_id=11 was mislabeled "Gait Physical
        // Examination" instead of "Movement Disorder Physical Examination").
        GAIT(1), UPPER_EXTREMITY(2), RUNNING(3), RESEARCH(4), MOVEMENT_DISORDER(11);

        private final Integer id;

        VisitKind(Integer id) {
            this.id = id;
        }

        public static VisitKind fromId(Integer id) {
            if (id == null) {
                return GAIT;
            }
            for (VisitKind kind : values()) {
                if (id.equals(kind.id)) {
                    return kind;
                }
            }
            return GAIT;
        }
    }

    public record Option(String key, String label, boolean enabled) {}

    private final VisitRepository visits;
    private final InterpretationRepository data;
    private final VisitImageService images;

    public InterpretationService(VisitRepository visits, InterpretationRepository data,
            VisitImageService images) {
        this.visits = visits;
        this.data = data;
        this.images = images;
    }

    public List<Option> options(int patientId, int visitId) {
        var visit = requireVisit(patientId, visitId);
        VisitKind kind = VisitKind.fromId(visit.getVisitTypeId());
        List<Option> options = new ArrayList<>();
        options.add(new Option("video", "Video Assessment", true));
        options.add(new Option("physical-exam", switch (kind) {
            case RUNNING -> "Running Physical Examination";
            case UPPER_EXTREMITY -> "UE Physical Examination";
            case MOVEMENT_DISORDER -> "Movement Disorder Physical Examination";
            default -> "Gait Physical Examination";
        }, true));
        options.add(new Option("kinematics", "Kinematics", hasKinematics(visitId, kind)));
        if (kind != VisitKind.UPPER_EXTREMITY) {
            // Legacy kinetics is file-based and applies only to gait, running and research visits.
            boolean kineticsType = kind == VisitKind.GAIT || kind == VisitKind.RUNNING
                    || kind == VisitKind.RESEARCH;
            options.add(new Option("kinetics", "Kinetics", kineticsType
                    && hasFirstConditionImage(patientId, visitId, "GroundReactionForces")));
            if (kind != VisitKind.RUNNING) {
                options.add(new Option("foot-model", "Foot Model",
                        data.hasMeasurements(visitId, Measurements.FOOT_MODEL)));
                options.add(new Option("emg", "EMG Assessment",
                        hasFirstConditionImage(patientId, visitId, "EmgWaveforms")));
            }
            options.add(new Option("pedobarograph", "Pedobarograph",
                    data.hasMeasurements(visitId, Measurements.FOOT_PRESSURE)));
            if (kind != VisitKind.RUNNING) {
                options.add(new Option("metabolic", "Metabolic (O2) Assessment",
                        data.hasMeasurements(visitId, Measurements.METABOLIC)));
                options.add(new Option("step-watch", "Step Watch",
                        !images.findInterpretationImages(patientId, visitId, "StepWatch").isEmpty()));
            }
        }
        options.add(new Option("photographs", "Photographs", true));
        options.add(new Option("assessment", "Assessment Form", true));
        options.add(new Option("recommendations", "Recommendations", true));
        options.add(new Option("signatures", "Signatures", true));
        return options;
    }

    private VisitRepository.InterpretationVisit requireVisit(int patientId, int visitId) {
        return visits.findInterpretationVisitByIdAndPatientId(visitId, patientId)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));
    }

    private boolean hasKinematics(int visitId, VisitKind kind) {
        if (kind == VisitKind.UPPER_EXTREMITY) {
            return data.hasMeasurements(visitId, Measurements.UPPER_EXTREMITY);
        }
        return data.hasMeasurements(visitId, Measurements.GAIT)
                || data.hasMeasurements(visitId, Measurements.BALANCE);
    }

    private boolean hasFirstConditionImage(int patientId, int visitId, String name) {
        return data.hasFirstCondition(visitId)
                && !images.findInterpretationImages(patientId, visitId, name).isEmpty();
    }
}











server:
  port: 8080
  servlet:
    session:
      timeout: ${SESSION_TIMEOUT:30m}
      cookie:
        http-only: true
        secure: ${SESSION_COOKIE_SECURE:true}
        same-site: lax

spring:
  application:
    name: gaitlab

  datasource:
    url: jdbc:postgresql://localhost:5432/gaittest_local
    username: postgres
    password: Hemanth@1008

  jpa:
    hibernate:
      ddl-auto: none
    open-in-view: false
    properties:
      hibernate:
        format_sql: true

logging:
  level:
    org.hibernate.SQL: debug

gaitlab:
  videos:
    root-path: ${GAITLAB_VIDEO_ROOT:./dev-data/PngFiles}




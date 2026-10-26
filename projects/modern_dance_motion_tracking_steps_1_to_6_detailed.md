## Step 1

A reference dataset of approximately 32 modern and contemporary dance videos performed by instructors or choreographers was assembled as an initial pilot collection. The instructor performances were captured with the ZED 2i stereo camera in its 1920 × 1080, 30-frames-per-second recording mode, preserving the stereo footage and synchronized frame timestamps for subsequent 3D pose extraction. Each recording was manually reviewed and divided into individual movement segments. Start and end timestamps and a descriptive movement label were assigned to each segment, yielding a target of approximately 145 annotated segments across 19 movement categories. The categories were defined from the instructor's choreography, with left- and right-direction variants distinguished where appropriate. Every segment remained linked to its original recording so the instructor's execution could serve as a reference for subsequent movement recognition and comparison. Because this initial collection provided relatively few examples per category, it was treated as a pilot dataset, with additional repeated demonstrations identified as necessary for classes with insufficient training or validation coverage.

## Step 2

The instructor recordings collected in Step 1 were processed offline using the existing YOLO-Pose and ZED 2i stereo-depth setup. The recorded stereo footage was replayed through the ZED SDK, allowing the same pose estimation and depth processing used for live video to be applied without requiring live analysis. YOLO-Pose detected 17 body keypoints in each frame, and the ZED 2i depth information was used to obtain their three-dimensional (x, y, z) coordinates. The extracted skeletons were saved with frame timestamps and keypoint confidence scores, while missing or unreliable detections were identified for review. To reduce differences caused by body size and camera distance, each skeleton was centered at the midpoint of the hips and scaled by torso length. The normalized 3D keypoint sequences were then divided using the movement start and end timestamps annotated in Step 1, preserving the corresponding movement labels and links to the original recordings. The unnormalized 3D coordinates were also retained to support analysis of movement across the stage. Because each reference recording contained a single instructor, multi-person identity tracking was not required for this stage.

## Step 3

The normalized 3D skeletal movement sequences obtained in Step 2 were used to adapt a pretrained SkateFormer model for modern and contemporary dance movement classification. Because SkateFormer was pretrained on NTU RGB+D skeletal data, the 17 YOLO-Pose keypoints were converted into the 25-joint NTU representation expected by its preprocessing pipeline. Twelve anatomically corresponding joints were mapped directly, while the remaining thirteen joints were approximated from the available 3D coordinates. The pelvis and spine positions were estimated using the hip and shoulder midpoints, and the neck and head were represented using torso-based estimates. Hand positions and fingertips were approximated by extending the forearm direction, while the ankles were retained as coarse placeholders for the unavailable foot landmarks. These estimated positions were treated as approximate features rather than measured joint locations. Each annotated movement sequence was sampled to 64 frames and arranged in SkateFormer's expected input format, with one person slot containing the instructor's skeleton and the second left empty. The preprocessing pipeline removed the unused spine-shoulder joint and organized the remaining 24 joints into the anatomical partitions used by the model. SkateFormer was initialized with pretrained action-recognition weights, and its original classification head was replaced with a 19-class output layer corresponding to the movement categories annotated in Step 1. The model was first trained with most of the pretrained backbone frozen, followed by selective fine-tuning to adapt its learned motion representations to modern and contemporary dance. To reduce overfitting, the initial training augmentation settings applied independently with a probability of 0.7 each were random 3D rotations about the vertical axis within ±15 degrees, simulated joint dropout at a probability of 0.05 per keypoint, temporal scaling of ±20% using interpolation, and Gaussian noise with σ = 0.01 on normalized 3D coordinates; the transformations were restricted to preserve the original movement labels. To reduce the mismatch between manually segmented training clips and continuous student performances, overlapping 64-frame windows were also extracted from the annotated instructor recordings, including windows near movement boundaries. Windows containing at least 80% frames from one labeled movement were assigned that dominant class for training, while more ambiguous mixed-movement windows were reserved for transition and unknown-movement evaluation rather than forced into one of the 19 classes. Recordings were divided into training and validation sets before window generation, ensuring that windows from the same recording did not appear in both sets. The adapted model was evaluated using classification accuracy, macro-F1, per-class confusion, and performance on held-out continuous sequences, with particular attention to boundary windows and classes involving approximated hand or foot landmarks.

## Step 4

Recorded performances of individual modern and contemporary dance students were processed offline using the same YOLO-Pose and ZED 2i setup employed for the instructor reference recordings. Each performance was captured with the ZED 2i stereo camera and replayed through the ZED SDK to extract 17 body keypoints and their corresponding 3D coordinates for each frame. The skeletal coordinates were normalized using the hip-centered, torso-scaled procedure established in Step 2 and converted from COCO-17 to the NTU-25 representation using the same joint-mapping and approximation rules applied during SkateFormer training. Because the pretrained SkateFormer architecture produces one movement classification per clip rather than a separate prediction for each frame, each student's continuous skeletal sequence was divided into overlapping 64-frame windows with an initial stride of 8 frames. Each window was prepared in SkateFormer's required input format and passed through the fine-tuned classifier to obtain probability scores for the 19 annotated movement categories. The class probabilities from all windows covering a given frame were averaged and then smoothed to estimate frame-level movement labels over the full recording. An initial maximum-probability threshold of 0.6 was applied to flag low-confidence frames as unknown (label -1), alongside frames with unreliable skeletal detections. These unknown intervals could represent transitions between movements, preparatory poses, non-choreographic actions such as standing or walking into position, or actions outside the 19 defined categories. Because a closed-set classifier could still assign high confidence to unfamiliar actions, the 0.6 threshold and smoothing settings were validated using manually labeled continuous performances containing transitions and out-of-category movements rather than being treated as reliable unknown detection by themselves. Unknown intervals were preserved in the output but excluded from downstream DTW comparison. Consecutive frames assigned the same movement label were grouped into segments, and their start and end times were recovered from the original recording timestamps. Short, isolated predictions were filtered to reduce false movement boundaries, and boundary accuracy was checked against manual annotations while tuning the window stride and smoothing parameters. The output consisted of automatically identified movement segments with predicted classes, estimated timestamps, and corresponding 3D skeletal coordinates. Recordings were initially limited to one student at a time to avoid reliance on persistent multi-person identity tracking; comparison with instructor reference movements was reserved for the subsequent stage.

## Step 5

The student movement segments identified in Step 4 were compared with the instructor reference movements collected in Step 1 using Fast Dynamic Time Warping (FastDTW). For each student segment, the predicted movement category was used to retrieve instructor demonstrations with the same label, and the corresponding normalized 3D skeletal sequences extracted in Step 2 were used for comparison. Unlike the NTU-25 skeletons reconstructed for SkateFormer classification, the DTW analysis used the original 17 YOLO-Pose keypoints with 3D coordinates to avoid introducing additional errors from approximated hand, fingertip, and foot landmarks. The student and instructor sequences were represented in the same joint order and normalized using the same hip-centered, torso-scaled procedure. At each candidate pair of student and instructor frames, the local pose cost was calculated as the mean 3D Euclidean distance over keypoints with valid detection and depth measurements in both frames. Pairs with too few reliable joints were excluded or flagged rather than treating missing coordinates as zeros. FastDTW aligned corresponding phases of the movement despite differences in execution speed, and the instructor demonstration with the lowest mean alignment cost was selected as the closest reference. The normalized DTW distance, d, was calculated by averaging the valid-joint pose costs over all matched frame pairs along the warping path, so the result accounted for both the number of comparable joints and the alignment-path length. A per-segment score was then calculated as Skeletal Similarity (%) = 100 × exp(−d / λ), where λ was a positive scaling parameter calibrated against both acceptable instructor demonstrations and student performances containing independently assessed movement deviations. This calibration distinguished ordinary variation from meaningful errors instead of treating every departure from a single reference as incorrect, and the score was not interpreted as an objective measure of artistic accuracy. The alignment path was retained to identify corresponding student and instructor frames, and joint-angle and coordinate differences at those matched frames were used to locate body regions and movement phases with the largest deviations. Because DTW absorbs differences in execution speed, the ratio of student-to-instructor movement durations was also recorded separately; where choreographic timing cues were available, movement-onset offsets were evaluated to preserve rhythm-related information. Segments with low classification confidence, insufficient valid skeletal data, or no suitable instructor reference were flagged for review instead of being assigned an unreliable similarity score. The output consisted of each student's identified movement, its closest instructor demonstration, the temporal alignment, calibrated skeletal similarity measurements, and localized differences for the subsequent LLM feedback stage.

## Step 6

The movement comparisons produced in Step 5 were used to generate personalized feedback for recorded modern and contemporary dance performances with **Google Gemini 3.8 Flash** (API model identifier: `gemini-3.8-flash`). Gemini was used as a text-based feedback generator rather than a pose detector or movement classifier; no raw video, reconstructed NTU joints, or unverified movement descriptions were required as model input. The feedback module first retrieved the predicted movement label and confidence from SkateFormer, the matched instructor reference and FastDTW alignment path, the calibrated Skeletal Similarity score, the student and instructor timestamps, and the original measured COCO-17 3D keypoint data. Corrections were considered only for classified segments with an available reference, sufficient stereo-depth and keypoint confidence, and deviations exceeding instructor-approved tolerances. An initial quality gate required at least 80% valid measured joint observations for the body region being discussed; deviations also had to persist for at least 0.25 seconds of the student's aligned movement, rather than arise from a single noisy frame. These settings were specified as initial design choices and were to be tuned on instructor-reviewed examples. Using the FastDTW frame correspondences, the preprocessing module calculated numerical comparisons such as elbow and knee angles, arm elevation, torso inclination, normalized joint displacement, and separately measured movement-duration or choreography-cue timing differences where available. Only measurements supported by the original ZED-derived 3D keypoints were retained, so imputed fingertip positions, toe orientation, and other reconstructed NTU joints were excluded from feedback evidence. The differences were ranked by persistence and magnitude relative to class-specific instructor tolerances, and a compact structured evidence record was created for each selected segment. The record included its identifier, movement category, reference demonstration, allowed stylistic variation, paired student/reference timestamps, classification confidence, skeletal similarity, region-validity statistics, and up to three measured deviations with numerical values and units. A fixed system prompt constrained Gemini to the supplied evidence and a movement-specific user prompt passed the structured record. The Gemini API requested a JSON response containing either an abstention or no more than two prioritized corrections, each identifying the affected body region, the specific measured difference, the corresponding frame timestamps, and a concise actionable instruction. A separate optional timing correction was generated only when an actual reference timing cue was provided. The JSON response was validated against its schema and cross-checked against the input evidence, including metric identifiers, numerical values, valid measurement regions, and referenced timestamps. Outputs containing unsupported claims or unverified corrections were rejected or sent for instructor review. Feedback records were then linked to the matching student and instructor frames for synchronized playback in Step 7. All feedback was generated **offline after the recorded performance had been analyzed**, not continuously from live video.

### Step 6: Reproducible feedback specification

**Model and API.** The feedback model was specified as Google **Gemini 3.8 Flash** (`gemini-3.8-flash`) through the Google GenAI API, with **text-only structured input**, no model fine-tuning, and a required `application/json` response constrained by a JSON schema. The model identifier and prompt version were saved with each feedback record for reproducibility. This selection is a project-specific implementation decision; it is not a claim about which Gemini version was used by another research team. Official model documentation: https://ai.google.dev/gemini-api/docs/models/gemini-3.8-flash ; structured-output documentation: https://ai.google.dev/gemini-api/docs/structured-output .

**Pre-LLM selection rules.** A movement proceeded to automated feedback only when (1) the SkateFormer movement label was not `-1` and its confidence was at least the Step 4 threshold of 0.6, (2) an instructor reference of the same category was available, (3) at least 80% of the relevant measured joint observations were valid, and (4) at least one joint or timing deviation exceeded its **instructor-approved, class-specific tolerance** for at least 0.25 seconds. These are initial candidate settings, not experimentally determined optimal thresholds. Similarity scores were used to prioritize review, not to infer a body-specific error without supporting joint measurements. A segment failing these gates received an `abstain` or `no_actionable_difference` status rather than invented coaching.

**System prompt (exact proposed template):**

```text
You are a modern and contemporary dance rehearsal feedback assistant.
You receive structured results from a 3D skeletal tracking and FastDTW comparison pipeline.
You do not view or analyze the performance video directly. Use ONLY the supplied data.

Rules:
1. Treat the instructor's stated movement goal and allowed variation as the reference.
   A difference is not automatically an error, especially in contemporary dance.
2. Give feedback only for the supplied evidence records that exceed the stated
   instructor-approved tolerance and have adequate measurement quality.
3. State the body region, specific metric, measured student and reference values,
   units, and aligned student/reference times for each correction.
4. Do not calculate new measurements, add unprovided observations, or speculate
   about emotions, effort, intent, injury, balance, hand shapes, finger position,
   foot orientation, or joint angles not present in the input.
5. Distinguish spatial pose differences from timing differences. Discuss rhythm
   or musical timing ONLY if an explicit reference timing cue is supplied.
6. Use no more than TWO prioritized, concise, actionable corrections per segment.
   If the data are uncertain, within tolerance, or insufficient, abstain.
7. Return valid JSON matching the supplied response schema. No markdown or prose
   outside that JSON. Use metric_id values present in the input exactly.
```

**User prompt (exact proposed template):**

```text
Evaluate one recorded student movement against its matched instructor movement.
Use the movement goal, permitted artistic variation, aligned measurement records,
quality flags, and optional timing cue in the JSON payload below.
For each correction, cite one supplied metric_id and report its paired measured
values and timestamps exactly as provided. Do not report more than two corrections.
If no supported correction is available, set status to "no_actionable_difference"
or "abstain" as appropriate and return an empty corrections array.

MOVEMENT_COMPARISON_JSON:
{{comparison_json}}
```

**Illustrative input record.** Numerical values here are synthetic examples showing the data contract, **not measured experimental results**. The system would construct this JSON after FastDTW; it would not ask Gemini to calculate the angles.

```json
{
  "segment_id": "student_007_segment_03",
  "reference_id": "instructor_004_arm_extension",
  "movement_class": "right_arm_extension",
  "movement_goal": "Extend the right arm to approximately shoulder height with a nearly straight elbow",
  "allowed_variation": "Small wrist and hand-style differences are acceptable",
  "student_interval_s": [4.2, 6.6],
  "reference_interval_s": [2.4, 4.6],
  "skateformer_confidence": 0.91,
  "skeletal_similarity_pct": 73.4,
  "valid_measured_joint_ratio": 0.95,
  "pose_source": "measured COCO-17 3D keypoints, not imputed NTU joints",
  "measurements": [
    {
      "metric_id": "right_elbow_angle_deg",
      "body_region": "right_arm",
      "student_value": 126.0,
      "reference_value": 173.0,
      "unit": "degrees",
      "student_time_s": 5.14,
      "reference_time_s": 3.12,
      "duration_above_tolerance_s": 0.38,
      "instructor_tolerance": "absolute difference <= 15 degrees",
      "valid": true
    },
    {
      "metric_id": "right_arm_elevation_deg",
      "body_region": "right_arm",
      "student_value": 68.0,
      "reference_value": 87.0,
      "unit": "degrees",
      "student_time_s": 5.22,
      "reference_time_s": 3.18,
      "duration_above_tolerance_s": 0.42,
      "instructor_tolerance": "absolute difference <= 10 degrees",
      "valid": true
    }
  ],
  "timing_cue": null
}
```

**Required JSON response contract.** The application validated these keys and types with a structured-output schema (all fields required; `null` permitted where shown):

```json
{
  "segment_id": "string",
  "movement_class": "string",
  "status": "feedback | no_actionable_difference | abstain",
  "corrections": [
    {
      "body_region": "string",
      "metric_id": "string",
      "student_value": 0.0,
      "reference_value": 0.0,
      "unit": "string",
      "student_time_s": 0.0,
      "reference_time_s": 0.0,
      "actionable_feedback": "string"
    }
  ],
  "timing_feedback": null,
  "review_reason": null
}
```

Here, `status` was restricted to the three listed values; the `corrections` array contained at most two objects, and `timing_feedback` and `review_reason` were either strings or `null`. The block above illustrates the response **shape**, rather than being a literal JSON Schema. The API was configured to constrain generation to the corresponding formal JSON Schema.

**Illustrative Gemini response:**

```json
{
  "segment_id": "student_007_segment_03",
  "movement_class": "right_arm_extension",
  "status": "feedback",
  "corrections": [
    {
      "body_region": "right_arm",
      "metric_id": "right_elbow_angle_deg",
      "student_value": 126.0,
      "reference_value": 173.0,
      "unit": "degrees",
      "student_time_s": 5.14,
      "reference_time_s": 3.12,
      "actionable_feedback": "Straighten the right elbow more as the arm extends; it was bent to 126° compared with the instructor's 173° at the aligned phase."
    },
    {
      "body_region": "right_arm",
      "metric_id": "right_arm_elevation_deg",
      "student_value": 68.0,
      "reference_value": 87.0,
      "unit": "degrees",
      "student_time_s": 5.22,
      "reference_time_s": 3.18,
      "actionable_feedback": "Raise the right arm closer to shoulder height; the measured elevation was 68° versus the instructor's 87°."
    }
  ],
  "timing_feedback": null,
  "review_reason": null
}
```

**Validation and evaluation.** The application rejected malformed JSON, unknown metric identifiers, numbers or timestamps that differed from the structured evidence, and corrections that referenced invalid or synthetic joints. Where `status` was not `feedback`, the corrections list was required to be empty. Unsupported natural-language claims were flagged for instructor review. Generated feedback was evaluated on held-out, instructor-annotated segments for (i) factual agreement with 3D measurements, (ii) correct attribution to body region and time, (iii) usefulness of the correction, and (iv) rate of unsupported or unnecessary feedback. These checks assessed feedback quality independently from SkateFormer's classification accuracy and the DTW similarity score.

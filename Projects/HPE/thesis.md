# Kumaresh.pdf — Complete capstone review preparation

RK, I’ve gone through Kumaresh.pdf, your thesis, and the project repository. Kumaresh.pdf is Guide 2: Every Figure in the Thesis. It covers all 12 thesis figures and tells you what to point at, what to say, and which questions a faculty member may ask. We’ll follow those 12 figures in order so that you can explain them confidently rather than memorising isolated descriptions.&#x20;

Kumaresh.pdf



The most important objective is to understand the story behind each figure: what problem it addresses, what evidence it presents, what the result actually proves, and what it does not prove.

## 1. Understand the project's story before the figures

[Image-based crop disease detection with federated learning | Scientific Reports](https://images.openai.com/static-rsc-4/bY88QwMz-9gcnKQqi3ueuyTFzr1BUPSBuUsEbiv3Hcp7DhgDvqLm8RzxuMW-hMUjCH93auGb5Y3lsvIjdplMiwYR6X4VWISeJiGicK9JiFnsnhtdBlwCbZZ72zVEBV2TihsfgJR6Rpcz8zLkZiWz3hvaD-oG4qE8YAIOA8xKqWg?purpose=inline)

Phase 1 — Training

Five simulated CCTV nodes learn from their own training data. They collaborate by exchanging updates to a small adapter, rather than sending their raw training images to a central server.

[Gst-nvtracker — DeepStream documentation](https://images.openai.com/static-rsc-4/sijEVHYsdyyBSO4mIwhlLbXfMPUfQmsoMZOJx0_Oh5_OyR8FUpNo_OD5_eX6KtpCVKAXXbyH6OjnD8eDnIax1Px-u3aUHmoLJ7sEgn2l2KZMznQ0gWF-v3xOtax_oWaBuTsufDVatDh-frMoE614mRw9radFJic5RRD6aAqI_94?purpose=inline)

Phase 2 — Searching for a missing person

A case is created, the nodes scan their own camera footage, and likely matches are sent to a local human reviewer. Only a confirmed sighting is sent upstream.

[Evaluating Federated Kolmogorov-Arnold Networks on Non-IID Data](https://images.openai.com/static-rsc-4/60mz3vTDRDUMafm9bn1IsjozSk-lvVKFCciw1nVds4d_3drV3OnR6AID2hljgbYKu1keYdbd39mZZOn3XH5tyvWCsY260tbRQB0gierkiwg6J4H-U7tHdHl9BhTjNOtt9WI3OwW5oC8FZJ_Op5KCLmQZ3R1fRaHadMiHbnfSIes?purpose=inline)

The main research finding

Federated training achieved approximately the same Rank-1 performance as centralised training, without pooling training data. However, the adapter did not measurably improve on the frozen ArcFace baseline in this dataset.

The thesis calls the system FedMPS — Federated Missing-Person Search. Its privacy approach combines federated learning, secure aggregation, differential privacy experiments, cancelable watchlist templates, local processing and human review. The repository provides the corresponding training, application, privacy, evaluation and result files.&#x20;

FedMPS_Capstone_Thesis.docx

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub

+1



### The key numbers to memorise

| Item                               | Value                   | What it means                                                  |
| ---------------------------------- | ----------------------- | -------------------------------------------------------------- |
| Camera nodes                       | 5                       | Five simulated camera sites                                    |
| VGGFace2 subset                    | 172,620 images          | The main training/evaluation source                            |
| Identities in the subset           | 8,631                   | Distinct people                                                |
| Recognition backbone               | 43.6 million parameters | Frozen ArcFace ResNet-50                                       |
| Federated adapter                  | 66,240 parameters       | The component trained across nodes                             |
| Adapter update                     | 0.26 MB                 | Communication per update, instead of 174.4 MB for the backbone |
| Default evaluation operating point | 3 × 3                   | Three enrolment photos and three fused probe frames            |
| Frozen backbone Rank-1             | 0.9474                  | 94.74% under the thesis's evaluation protocol                  |
| Federated adapter Rank-1           | 0.9473                  | 94.73% under that protocol                                     |
| Centralised adapter Rank-1         | 0.9476                  | 94.76% under that protocol                                     |

These values come from the thesis and the repository's reported results. Be precise: 94.74% is not a promise of 94.74% accuracy in every real-world CCTV deployment. It is the measured Rank-1 result under a specified evaluation protocol.&#x20;

FedMPS_Capstone_Thesis.docx

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub



# 2. Explain all 12 figures in Kumaresh.pdf

## Figure 1 — Workflow of FedMPS

Kumaresh.pdf: page 1 · Thesis: Section 2.1, printed page 13

[IASC | Free Full-Text | Person Re-Identification with Model-Contrastive Federated Learning in Edge-Cloud Environment](https://images.openai.com/static-rsc-4/cMdCvab8pM-6EH2KbYoyJz06s-TqusTNgz7kzEZR_weJI1efv-ezOxxSAevAKq8dInDrYUEyXKNk-f4QnECS83iGcDGEWc_zA5_sEXN3dl1Cx9pWZAYl8hnnzWahFZ4dJXRw2m0z2RpaoMKeLUwlwmoG78yUCF8-FacACJF0N9g?purpose=inline)

[techscience.com](https://www.techscience.com/iasc/v38n1/55288/html?utm_source=chatgpt.com)

### What the figure means

The diagram contains two main workflows.

Top: Training phase

1. Start with VGGFace2 face images.
2. Align the faces so that their facial landmarks match a consistent template.
3. Apply simulated CCTV degradation.
4. Generate face embeddings using the pretrained ArcFace backbone.
5. Partition the training identities among five nodes.
6. Federate training of the small adapter to obtain the final adapter checkpoint.

Bottom: Operational phase

1. Open a missing-person case.
2. Generate node-specific watchlist templates.
3. Scan video at the camera site.
4. Detect, track, embed and match faces.
5. Combine match evidence over several frames.
6. Send a pending alert to the local operator.
7. Record the decision and audit trail; send a confirmed sighting upstream.

The diagram's downward arrow is important: the adapter trained in the first phase becomes part of the recognition system used in the second phase.&#x20;

Kumaresh.pdf

FedMPS_Capstone_Thesis.docx



### What to say to your faculty guide

"Figure 1 gives the complete architecture of FedMPS. It has two phases: federated training and operational missing-person search. During training, five nodes learn a shared adapter without pooling their training images. During operation, each node processes its camera footage locally, checks candidate matches, and requires human confirmation before sending a sighting to the server. The audit log records the actions taken."

Copy presentation script

Likely viva question: Where is the privacy in this architecture?

The protection happens at multiple points: training uses model updates instead of raw training images; watchlist templates can be keyed differently for each node; raw camera video and non-matching face crops stay local; and a sighting is sent upstream only after human confirmation.

An important qualification: the 64-byte template is not the only data necessarily sent for a case. In the thesis's default cosine-matching mode, the node also receives an enrolled embedding. Template mode is the option designed to use protected codes only. Do not claim every mode has precisely the same data flow.&#x20;

FedMPS_Capstone_Thesis.docx



Where to look in the repository: `training/01_prepare_data.py`, `training/02_run_experiments.py`, the `fedmps/fl/` modules, `application/run_video_scan.py`, and the server/node console files. The repository's [README and run guide](https://github.com/Lokesh2003GN/fedmps-capstone/blob/main/HOW_TO_RUN.md?utm_source=chatgpt.com) provide the code map.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub

+1



## Figure 2 — Node split of VGGFace2 and the pipeline log

Kumaresh.pdf: page 2 · Thesis: Section 2.2.1, printed page 15

### What the figure means

The figure contains two kinds of evidence:

- Left: Dataset partition. It shows how many training and validation images are allocated to the five nodes.
- Right: Data-preparation log. It shows that the image-alignment process was actually run and records the number of identities held out.

The nodes are deliberately unequal. Node 0 contains 258 training-node identities, while Node 4 contains 779. Each node represents a camera site with a different amount of data.&#x20;

Kumaresh.pdf

FedMPS_Capstone_Thesis.docx



### The counts you should know

| Dataset component                      | Identities                      | Images       |
| -------------------------------------- | ------------------------------- | ------------ |
| Whole VGGFace2 subset                  | 8,631                           | 172,620      |
| Held-out evaluation identities         | 6,042                           | 120,840      |
| Identities allocated across five nodes | 2,589                           | 51,780 total |
| Node training data                     | —                               | 41,420       |
| Node validation data                   | 518 identities across the nodes | 10,360       |

The 6,042 held-out evaluation identities are further divided into the 3,021-person watchlist and 3,021 people who are not on the watchlist.

Don't mix up evaluation identities and validation identities. The 6,042 evaluation identities are separate from the approximately 518 identities withheld at the five training nodes for early stopping. Those two held-out sets serve different purposes.&#x20;

FedMPS_Capstone_Thesis.docx



### What to say

"The dataset is split by identity, rather than randomly splitting individual images. The five nodes have unequal amounts of training data to approximate different camera sites. The validation identities are kept out of local training so we can detect overfitting, while the separate evaluation identities are used to measure final performance."

Likely question: Why split by identity instead of image?

Because the same person's images appearing in both training and testing can make the evaluation artificially easy. The model could benefit from having already seen that person instead of genuinely generalising to unseen identities.

Likely follow-up: What does non-IID mean here?

It means the nodes' data distributions are not identical. Each node has a different camera profile, and therefore systematically different image degradation. This is a realistic challenge for federated learning.

## Figure 3 — Aligned face crops and CCTV degradation

Kumaresh.pdf: page 2 · Thesis: Section 2.2.2, printed page 16

### What the figure means

The figure visually compares clean enrolment photos with degraded probes, and also shows the same identity as seen through different simulated camera profiles.

The degradation chain approximates the problems caused by CCTV:

1. Motion blur.
2. Downscaling to a small effective face size.
3. Lighting or exposure changes.
4. Sensor noise.
5. JPEG compression.
6. Upscaling to the working image size.

Upscaling restores the dimensions, but it cannot restore the facial detail that was lost.

For example, the guide shows one camera profile producing approximately 24-pixel faces at JPEG quality 40, while another produces approximately 44-pixel faces at quality 57. That persistent difference between nodes is another illustration of non-IID data.&#x20;

Kumaresh.pdf

FedMPS_Capstone_Thesis.docx



### The number to remember

The thesis reports that the average cosine similarity between two clean images of the same person is 0.514, falling to 0.219 when comparing a clean image with a degraded one.

This is a similarity score, not Rank-1 accuracy. The purpose of the comparison is to demonstrate how much degradation affects the facial representation before we discuss model training.

### What to say

"Figure 3 shows why missing-person identification is difficult. The reference photo may be clear, but the camera image can be tiny, blurred, noisy and badly exposed. Each simulated node has its own camera profile, which creates heterogeneous data. The average same-person cosine similarity drops from 0.514 for clean images to 0.219 for clean-versus-degraded pairs."

Likely question: Why not simply train the whole ArcFace model again?

The backbone has 43.6 million parameters, while each node has relatively limited local training data. The project freezes that pretrained backbone and trains a much smaller adapter instead.

Where to look in the repository: `fedmps/data/degrade.py` and the data-preparation script `training/01_prepare_data.py`.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub



## Figure 4 — Derivation of a cancelable watchlist template

Kumaresh.pdf: page 3 · Thesis: Section 2.2.11, printed page 21

This is one of the most important figures because it connects your privacy design to a concrete technical mechanism.

### Understand the process step by step

Reference photograph

Processed on the server

ArcFace embedding

512 numerical values

Binarisation

Convert the embedding into 512 bits

Node 0

Key 0 → 64 bytes

Node 1

Key 1 → 64 bytes

Node 2

Key 2 → 64 bytes

Nodes 3 and 4

Their own keys, separately

The process is:

1. Generate the 512-dimensional ArcFace embedding of the reference face.
2. Binarise the embedding into a 512-bit representation.
3. Derive a key specific to the case and node, using the project’s master secret, case identifier, node number and SHA-256.
4. XOR the bits with the key.
5. Pack the resulting 512 bits into 64 bytes and issue the protected template to that node.

The live face at a node is processed into a comparable representation using the corresponding node key. The similarity is computed from the Hamming distance between the binary codes.&#x20;

Kumaresh.pdf

FedMPS_Capstone_Thesis.docx



### What is XOR, and why does it allow matching?

XOR is a bitwise operation:

| A | B | A XOR B |
| - | - | ------- |
| 0 | 0 | 0       |
| 0 | 1 | 1       |
| 1 | 0 | 1       |
| 1 | 1 | 0       |

The useful property is that applying the same key to both codes preserves their Hamming distance:

\\[ d_H(x\oplus k,\ y\oplus k)=d_H(x,y) \\]

Here, \\(d_H\\) counts how many bit positions differ.

This lets two protected codes remain comparable without exposing the original binary codes on disk or over the network. In the implementation described by Kumaresh.pdf, the matching code also supports an equivalent operation by removing the key from the stored template in memory before comparison.&#x20;

Kumaresh.pdf



### What to say

"Figure 4 explains how we protect the missing person's watchlist representation. The server converts the reference face embedding into a binary code and XORs it with a node-specific key. The result is a 64-byte template. Different nodes receive differently keyed templates, which makes the templates harder to link across sites, while the matching process uses Hamming similarity."

Likely question: If every node has a different template, how can they search for the same person?

Each node matches a live face using the key and representation corresponding to its own template. The matching operation compares the live representation against that node's protected representation.

Likely question: Does this make the system completely secure?

No. It protects stored and transmitted templates and makes cross-node linking harder. It is not a guarantee against a fully compromised node that possesses its own key and can inspect its local computations.

Another important distinction: cancelable templates and differential privacy are different mechanisms. Template protection applies to the watchlist representation; differential privacy attempts to bound the information leaked through training updates. The thesis reports that no differential-privacy setting was found that offered both useful privacy and acceptable accuracy.&#x20;

FedMPS_Capstone_Thesis.docx



Repository paths to know: `fedmps/privacy/template.py` and `fedmps/pipeline/match.py`.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub



## Figure 5 — Operational pipeline at a camera node

Kumaresh.pdf: page 3 · Thesis: Section 2.2.14, printed page 23

### The eight stages

1\. Frames

Sample video at up to 8 frames per second.

2\. Detect

SCRFD finds faces and facial landmarks.

3\. Track

ByteTrack links detections belonging to the same person over time.

4\. Quality gate

Reject faces that are too small, blurred, poorly exposed or too far turned sideways.

5\. Embed

Create a 512-dimensional face representation with the frozen backbone and adapter.

6\. Match

Compare with the watchlist using cosine or template matching.

7\. Vote

Require a match in 3 of the last 5 frames of the same track.

8\. Pending alert

Create one alert for local human review, rather than sending frame-level hits directly to the server.

### Important thresholds

| Setting                       | Value                  |
| ----------------------------- | ---------------------- |
| Maximum frame sampling rate   | 8 fps                  |
| Minimum distance between eyes | 24 pixels              |
| Accepted mean brightness      | 25–235                 |
| Maximum head yaw              | 45°                    |
| Cosine-mode threshold         | 0.24                   |
| Template-mode threshold       | 0.2656                 |
| Voting criterion              | 3 of the last 5 frames |

The two matching modes have different score scales and thresholds. Do not compare their raw scores as though they were directly interchangeable.&#x20;

Kumaresh.pdf

FedMPS_Capstone_Thesis.docx



### Why have both a quality gate and track voting?

They control different sources of error.

- Quality gate: rejects inputs that are too unreliable to recognise safely.
- Track voting: prevents a single accidental frame-level match from generating an alert.

For example, a face that is just a few pixels wide may yield an embedding, but that embedding is unlikely to be reliable enough to identify a person. Rejecting it can be better than making a confident but incorrect match.

### What to say

"Figure 5 shows the full processing pipeline at each node. We detect and track faces, reject poor-quality observations, generate embeddings, compare them with the watchlist and require agreement across multiple frames before raising an alert. This reduces false alarms while keeping the video processing local."

Likely question: Why use 3 out of 5 frames instead of accepting one match?

A single frame can be corrupted by blur, pose or noise. Agreement across several frames provides stronger evidence and reduces the effect of isolated false matches.

Repository paths: `fedmps/pipeline/detect.py`, `fedmps/pipeline/quality.py`, and `fedmps/pipeline/match.py`.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub



## Figure 6 — Node console: human review of a pending alert

Kumaresh.pdf: page 3–4 · Thesis: Section 2.2.15, printed page 24

### What the screenshot demonstrates

The screenshot shows the node operator's review console. The operator sees two images side by side:

- The reference thumbnail associated with the missing-person case.
- The candidate face observed by the local camera.

In the demonstration, the match score is 0.552, and three of three frames in that track agree. The reviewer can choose Confirm or Reject.&#x20;

Kumaresh.pdf



### The crucial security property

The software does not automatically turn every model match into a confirmed sighting.

A pending alert stays in the node's review queue. The operator makes the decision, and only a confirmation causes the candidate crop to be sent upstream. Opening the reference thumbnail is also recorded in the audit log.

The reference thumbnail is available for comparison; this does not mean that the full-resolution reference photograph needs to be distributed to every node. The thesis says that the server retains a 200 × 200 thumbnail for human review rather than storing the full reference photo.&#x20;

FedMPS_Capstone_Thesis.docx



### What to say

"Figure 6 demonstrates the human-in-the-loop safeguard. A model match creates a pending alert, not an automatic disclosure. The local operator compares the reference thumbnail with the detected face and confirms or rejects the candidate. Only a confirmed alert sends the crop upstream, and the decision is audited."

Likely question: Why is human review necessary if the model already has a match score?

A score is probabilistic evidence, not a guarantee that the identity is correct. Human review provides an additional false-positive check and establishes accountability for any disclosure.

Likely question: What happens when the operator rejects the alert?

The candidate is rejected locally and is not sent upstream as a confirmed sighting. The review decision is recorded, and the thesis also describes confirms and rejects as possible training labels for future work.

Repository path: `application/node_app.py`.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub



## Figure 7 — Audit log after a completed case

Kumaresh.pdf: page 4 · Thesis: Section 2.2.15, printed page 25

### Understand the ten log rows

The screenshot contains ten audit rows. Read them from the bottom upwards to reconstruct the chronological workflow:

| Step | Action               | Meaning                                                         |
| ---- | -------------------- | --------------------------------------------------------------- |
| 1    | `case.create`        | The server opens a case with a 30-day default expiry.           |
| 2–6  | `template.issue` × 5 | A different keyed template is issued to each of the five nodes. |
| 7    | `alert.raise`        | Node 2 raises a candidate alert.                                |
| 8–9  | `reference.view` × 2 | The operator opens the reference thumbnail twice.               |
| 10   | `alert.confirmed`    | The operator confirms the sighting.                             |

The log records the actor, action and time. The project's audit interface supports both server-side and node-side oversight.&#x20;

Kumaresh.pdf

FedMPS_Capstone_Thesis.docx



### What to say

"Figure 7 demonstrates traceability. We can reconstruct the complete chain from opening the case to issuing the five templates, raising a candidate alert, viewing the reference and confirming the sighting. This allows the system to record who performed each sensitive action and when."

Likely question: What is the practical value of an audit log?

If somebody asks why an alert was confirmed, who accessed the reference image, or whether a case was closed, the system has a record to inspect. Auditability is important when working with sensitive biometric information.

Likely question: Is the audit log just for debugging?

No. It supports operational oversight and accountability, not merely debugging. It provides a record of access, review and disclosure decisions.

Repository path: `fedmps/store.py`, which stores audit-related records alongside the case, template and alert data.&#x20;

Kumaresh.pdf



## Figure 8 — Rank-1 at each step of the project

Kumaresh.pdf: page 4–5 · Thesis: Section 4.1, printed page 30

This figure is important because it tells the story of the experiments before the adapter result is discussed.

### The values shown in the figure

| Experimental stage                         | Rank-1 |
| ------------------------------------------ | ------ |
| CelebA, unaligned faces, 1 photo × 1 frame | 0.4807 |
| Add the scoring layer                      | 0.5029 |
| Move to the 3 × 3 operating point          | 0.9186 |
| Use VGGFace2 and ArcFace alignment         | 0.9848 |
| Increase the watchlist to 3,021 people     | 0.9474 |



Kumaresh.pdf

FedMPS_Capstone_Thesis.docx



### How to interpret the chart

Rank-1 measures how often the correct identity is ranked first under the specified evaluation setup.

The first four stages show improvements from changes to the representation and evaluation process. Adding more enrolment photos and combining multiple probe frames supply more evidence for each identity. Face alignment makes the input more consistent with the way ArcFace was trained.

The final Rank-1 value falls from 0.9848 to 0.9474 because the watchlist grows to 3,021 people, making the search harder. More candidates create more opportunities for a wrong identity to rank first.

### One critical caveat

This is a progression chart, not a perfectly controlled experiment in which only one variable changes at every step. The transition involving VGGFace2 and alignment changes both the dataset and alignment setup.

For a controlled alignment result, the thesis separately reports an A/B test on the same 5,000 identities: Rank-1 increased from 0.206 to 0.985 with alignment. That controlled comparison is the stronger evidence for the effect of alignment alone.&#x20;

FedMPS_Capstone_Thesis.docx



### What to say

"Figure 8 shows how the project developed from a weak initial baseline to a much stronger recognition pipeline. Using multiple enrolment images and fused frames and correcting face alignment were important improvements. Enlarging the watchlist to 3,021 people then made the identification task harder, so Rank-1 fell to 0.9474. These results establish the baseline against which we test the federated adapter."

Likely question: Why does the last bar decrease?

Because the number of people in the watchlist increased. The system must now distinguish the correct person from many more candidates.

Likely question: Can you attribute all the improvement from 0.9186 to 0.9848 to alignment?

No. The dataset and alignment setup change together in that progression. The separate A/B test on the same identities is the comparison used to isolate the alignment effect.

## Figure 9 — Rank-1 across enrolment/frame settings and the scoring layer

Kumaresh.pdf: page 5 · Thesis: Section 4.1, printed page 30

This figure has two panels.

### Left panel: number of enrolment photos and fused frames

The panel compares configurations using different numbers of reference photos and probe frames. The general trend is that adding useful observations improves Rank-1.

- An enrolment photo is a reference image used to represent the missing person.
- A probe frame is an image observed by a camera during the search.
- Frame fusion combines evidence from several frames to produce a more robust representation of the observed person.

The project's default configuration is three enrolment photos × three fused probe frames.

### Right panel: the scoring-layer improvements

At the 3 × 3 operating point, the guide reports approximately:

| Scoring configuration   | Rank-1 |
| ----------------------- | ------ |
| Raw cosine similarity   | 0.929  |
| Plus centring           | 0.936  |
| Plus partial rotation   | 0.939  |
| Plus hubness correction | 0.947  |



Kumaresh.pdf

FedMPS_Capstone_Thesis.docx



The three corrections work as follows:

Centring: removes an average embedding offset associated with the camera or degraded-image distribution.

Partial rotation: estimates a geometric rotation between degraded and clean embeddings and applies part of that correction to the probes.

Hubness correction: reduces the advantage of gallery entries that appear unusually similar to many unrelated probes.

The scoring layer improves identification without requiring the adapter to learn all of those corrections.

### What to say

"Figure 9 shows two sources of improvement. On the left, using additional reference photographs and multiple frames provides better identity evidence. On the right, centring, partial rotation and hubness correction progressively improve Rank-1 from about 0.929 to 0.947. Importantly, the same scoring layer is applied to the frozen, local, federated and centralised arms, so we do not attribute its gains to the adapter."

Likely question: Does this prove the federated adapter adds 1.8 percentage points?

No. The 1.8-point improvement is the approximate difference between the raw cosine baseline and the complete scoring layer. It is not the gain due to federated training. Every compared arm receives the same scoring corrections.&#x20;

FedMPS_Capstone_Thesis.docx



Repository paths: `fedmps/eval/metrics.py`, `fedmps/eval/calibrate.py` and `fedmps/eval/experiments.py`.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub



## Figure 10 — Verification ROC, identification DET and closed-set CMC

Kumaresh.pdf: page 5 · Thesis: Section 4.2, printed page 31

This figure is useful because it shows that the result is not based on just one accuracy number. The three panels measure different aspects of recognition performance.

### 1. ROC — Receiver Operating Characteristic

ROC evaluates face verification: deciding whether two face images belong to the same person.

It plots the true acceptance rate against the false acceptance rate. A better ROC curve generally stays higher, meaning the system accepts more genuine matches for the same false-acceptance rate.

### 2. DET — Detection Error Trade-off

The DET plot shows the trade-off between missed identifications and false identifications at different thresholds. Lower error rates are better.

It helps answer a practical question: how many genuine missing-person matches might be missed when the system is configured to limit false alerts?

### 3. CMC — Cumulative Matching Characteristic

CMC measures how often the correct identity appears within the top \\(k\\) results. For example, Rank-1 asks whether it appears first, while Rank-5 asks whether it appears among the first five candidates.

### What the figure actually tells us

The clean-probe curve performs substantially better than the CCTV-degraded results. The frozen, centralised and federated curves largely overlap, while the naïve local-only arm performs worse in the plotted comparison.&#x20;

Kumaresh.pdf

FedMPS_Capstone_Thesis.docx



That is the central observation: the degradation caused by the camera is a much larger problem than the choice between the frozen, centralised and federated models in these experiments.

### What to say

"Figure 10 compares recognition across three evaluation views: verification, open-set identification errors and closed-set ranking. The frozen, centralised and federated models have nearly overlapping curves, which means their performance is similar over a range of operating points, not just at one threshold. The much larger difference is between clean and degraded probes."

Likely question: Why present three curves or metrics instead of only Rank-1?

Each measures a different property. ROC evaluates verification, DET helps assess threshold-dependent identification errors, and CMC measures ranking performance. Together they provide a broader view of the system.

An important viva nuance: the lower local-only curve is the naïve local-training baseline. The thesis later performs a fairer comparison by applying the same early-stopping and FedProx regularisation. That local-only model reaches Rank-1 0.9473, matching the federated result. Therefore, the fair conclusion is not that federated training always beats local-only training; it is that federated training matches the centralised baseline without pooling training data.&#x20;

FedMPS_Capstone_Thesis.docx

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub



## Figure 11 — Validation Rank-1 across federated rounds

Kumaresh.pdf: page 5–6 · Thesis: Section 4.4, printed page 33

### What is a federated round?

One round consists of the server sending the global adapter to the nodes, each participating node training locally, and the server aggregating the resulting updates into a new global adapter.

The experiment runs for 30 rounds. After each round, the validation performance is measured.

### How to interpret the curve

The figure shows:

- Frozen-backbone validation Rank-1: 0.9486.
- Best validation result: 0.9514 at round 1.
- Performance subsequently declines, reaching around 0.9475 at round 18.

The key mechanism is early stopping. Rather than automatically taking the final round, the training loop remembers the round with the highest validation Rank-1. In the curve shown, that is round 1.&#x20;

Kumaresh.pdf

FedMPS_Capstone_Thesis.docx



### Why would accuracy decline after training longer?

The adapter is trained on relatively few identities per node. It can learn a useful correction quickly, but further training can cause it to overfit the local training identities. Training loss can continue to improve even while performance on unseen people deteriorates.

Validation-based model selection helps prevent that.

### What to say

"Figure 11 shows the importance of early stopping. Validation Rank-1 peaks at 0.9514 in the first federated round and then declines as training continues. This suggests that the small adapter learns what it can very quickly and subsequently begins to overfit. Therefore, we keep the best validation checkpoint instead of automatically using the last round."

Likely question: Why does the thesis mention round 26 on AWS if this figure selects round 1?

The thesis reports small numerical differences between the AWS and laptop runs. Floating-point differences changed the scores for a few probes, and the AWS run selected round 26 while the laptop run selected round 1. The choice of round was sensitive to only a few probes, demonstrating that the adapter's measured effect is very small.&#x20;

FedMPS_Capstone_Thesis.docx



Very important distinction: the 0.9514 and 0.9486 values are from the validation curve. Do not compare them directly with the 0.9473 federated test Rank-1 as though they are measurements on the same set of identities.

## Figure 12 — Annotated CCTV frame from Node 2

Kumaresh.pdf: page 6 · Thesis: Section 4.8, printed page 37

This is real output from the demonstration pipeline, rather than a conceptual architecture diagram.

### What you see in the frame

- The red bounding box labelled `#1 +0.71` identifies the candidate person, associated with track 1.
- The score `+0.71` is the matching score for that particular frame.
- Grey boxes labelled `too_small` indicate people rejected by the quality gate.
- The footer states that raw video never leaves the device.

The other faces are not automatically identified merely because the detector sees them. They must first pass the quality gate and the matching and voting stages.&#x20;

Kumaresh.pdf



### What to say

"Figure 12 shows the pipeline working on an actual video frame. The red box indicates a matching candidate, while the grey boxes show faces rejected as too small to recognise reliably. The purpose of the quality gate is to avoid producing unreliable matches from poor-quality faces, and the raw video remains local to the node."

Likely question: Why is the score 0.71 in this frame but 0.55 in the review console?

The values represent different things. The 0.71 is the score for an individual frame, whereas the approximately 0.55 score in the review console is the alert-level score associated with the track. They should not be assumed to be the same measurement.&#x20;

Kumaresh.pdf



Likely question: Does this image demonstrate real-world deployment across five independent camera networks?

No. It demonstrates the operational pipeline's output. The thesis describes the five-node federated setup as simulated; the five nodes run inside one process on one machine. A fully distributed network deployment remains future work.&#x20;

FedMPS_Capstone_Thesis.docx



# 3. The questions your faculty guide is most likely to ask

These are the questions I would practise first because they test whether you understand the research rather than just the diagrams.

## Q1. What is the main contribution of FedMPS?

Answer: FedMPS combines privacy-preserving federated learning with an operational missing-person search pipeline. The five nodes can collaborate without pooling training images, the watchlist can be distributed using node-specific protected templates, and human review and audit logging control and record disclosures.

## Q2. If the adapter gives no improvement, why is this project useful?

Answer: The main demonstrated benefit is that federated training matches centralised training while keeping the training data at the nodes. On this dataset, the adapter itself provides no measurable Rank-1 improvement over the frozen backbone. That is a valid experimental finding. It means the project demonstrates the privacy-preserving training and search workflow, not a universal improvement in recognition accuracy.

The measured test Rank-1 values are 0.9474 for the frozen backbone, 0.9473 for the federated adapter and 0.9476 for the centralised adapter.&#x20;

FedMPS_Capstone_Thesis.docx



## Q3. What is the difference between federated learning and centralised training?

Answer: In centralised training, the training data from all sites is pooled for training. In federated learning, each node trains on its own data and sends model updates to an aggregator, which combines the updates into the global model.

The purpose is to obtain collaborative training without requiring raw training data to be pooled at a single server.

## Q4. What is the difference between the frozen backbone and the adapter?

Answer: The ArcFace backbone is the pretrained face-recognition network with 43.6 million parameters. It produces a 512-dimensional embedding and remains frozen in this project. The adapter is a small residual MLP with 66,240 parameters that is trained to adjust degraded-face embeddings.

Its size reduces the update from 174.4 MB to approximately 0.26 MB per round.&#x20;

FedMPS_Capstone_Thesis.docx



## Q5. What is the difference between Rank-1 and cosine similarity?

Answer: Cosine similarity measures how similar two embedding vectors are. It is a score used for comparing faces.

Rank-1 is an identification metric: it measures how frequently the correct identity ranks first among the candidates under a defined evaluation protocol. A cosine score of 0.71 is not the same as 71% accuracy.

## Q6. Why is early stopping necessary?

Answer: The small adapter can overfit the limited identities available at the nodes. The training loop scores the model on held-out validation identities after each round and keeps the round with the best validation Rank-1. In the illustrated convergence run, the best round is round 1.

## Q7. Are all privacy mechanisms equally successful?

Answer: No. The project evaluates several mechanisms with different outcomes:

- Secure aggregation: pairwise masks protect individual updates while the server aggregates them. The reported masked and unmasked runs have identical Rank-1, 0.9376, with a measured runtime overhead of approximately 0.8%.
- Differential privacy: implemented on the model updates, but no operating point was found that was both meaningfully private and usable at the required accuracy.
- Cancelable templates: protect watchlist representations and make templates less linkable across nodes, but come with an accuracy trade-off.

These are measured results, not interchangeable guarantees.&#x20;

FedMPS_Capstone_Thesis.docx

[image](https://www.google.com/s2/favicons?domain=https://github.com\&sz=32)

GitHub



## Q8. Why is the human reviewer part of the architecture?

Answer: Face recognition produces candidates, not guaranteed truths. The local operator reviews the candidate, makes the confirmation decision and creates a record of the action. This reduces automatic false disclosures and supports accountability.

## Q9. Is the system production-ready?

Answer: It is an end-to-end prototype with a demonstrated training pipeline and working consoles. However, the federated nodes are simulated within one process, and the two consoles use a shared local SQLite store. A real deployment would need independent node stores, network communication, stronger production deployment controls and appropriate edge hardware.&#x20;

FedMPS_Capstone_Thesis.docx



## Q10. What is the single most important result to communicate honestly?

Answer: Federated training achieved approximately the same Rank-1 performance as centralised training, but the learned adapter did not measurably improve on the frozen backbone for this dataset. We therefore demonstrate privacy-preserving collaboration and an operational search workflow without claiming that federated learning inherently improves face-recognition accuracy.

# 4. Important mistakes to avoid during the review

| Avoid saying                                                          | Say this instead                                                                                                             |
| --------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| "The federated model is more accurate than the centralised model."    | "The federated and centralised results are effectively equivalent under our evaluation."                                     |
| "The adapter improves Rank-1 by 1.8 points."                          | "The approximately 1.8-point improvement comes from the scoring layer, which is applied to every comparison arm."            |
| "Every node receives only a 64-byte code."                            | "Template mode uses protected codes; in the default cosine mode, the node also receives an enrolled embedding."              |
| "The 6,042 validation identities select the best round."              | "The 6,042 held-out evaluation identities are separate from the node validation identities used for early stopping."         |
| "The result proves real-time performance on every CCTV network."      | "The measured performance depends on the hardware, video, threshold and evaluation setup."                                   |
| "The five nodes are deployed on separate machines."                   | "The five-node training setup is simulated; a real distributed deployment is future work."                                   |
| "Differential privacy protects the system without reducing accuracy." | "Differential privacy is implemented and evaluated, but the project did not find a usable privacy–accuracy operating point." |

These distinctions follow the thesis's reported methodology, limitations and measured results.&#x20;

FedMPS_Capstone_Thesis.docx



# 5. A two-minute explanation of Kumaresh.pdf

Practise this as one continuous explanation. You can use it to introduce the figures before the guide asks you to explain them individually.

"Kumaresh.pdf explains the 12 figures in our FedMPS thesis, covering the architecture, dataset, privacy mechanisms, operational pipeline and experimental results.

Figure 1 shows the two phases: federated training and operational missing-person search. Figures 2 and 3 explain how the VGGFace2 dataset is partitioned across five nodes and why different CCTV conditions make recognition difficult.

Figure 4 demonstrates cancelable watchlist templates. A reference image is converted into an embedding, binarised and keyed differently for each node. Figure 5 presents the node's processing pipeline, from detecting and tracking faces to checking quality, matching and voting across frames. Figures 6 and 7 then demonstrate the human-review workflow and the audit trail.

The result figures explain how the system was evaluated. Figure 8 shows the progression of the recognition baseline. Figure 9 demonstrates the benefits of multiple enrolment photos, fused frames and the scoring layer. Figure 10 compares verification and identification metrics across training methods. Figure 11 demonstrates why early stopping is necessary, and Figure 12 shows an actual annotated frame from the operational pipeline.

The key conclusion is that federated training matches centralised training while retaining training data locally. However, the adapter does not measurably improve on the frozen backbone in the main evaluation. The project therefore demonstrates a privacy-preserving collaboration and search workflow, with its limitations reported honestly."

Copy two-minute explanation

## Final preparation checklist

Before the review, make sure you can explain these without reading your notes:

- The two phases of FedMPS and why they are separate.
- The difference between the frozen ArcFace backbone and the federated adapter.
- The dataset split, non-IID data and the distinction between validation and test identities.
- How XOR-based cancelable templates work and what they do not protect against.
- The eight operational pipeline stages and their thresholds.
- The purpose of the human-review console and audit log.
- Rank-1, ROC, DET, CMC, cosine similarity and early stopping.
- Why the adapter's lack of measurable gain must be reported honestly.
- What is demonstrated in the prototype and what remains future work.

The best next step is a faculty-style mock viva, one figure at a time. Start with Figure 1: explain the architecture aloud without reading the script, and then test yourself on the privacy, training and deployment questions. That will reveal which parts you understand and which need another pass before moving on to Aswin.pdf.



# Aswin.pdf — Complete capstone review preparation

RK, Aswin.pdf is Guide 4: Federated Learning Infrastructure, Edge Systems and Dashboards. Unlike Kumaresh.pdf, which concentrated on explaining thesis figures, this guide is focused on how the system works internally and how to demonstrate the application.

It has two parts:

- Part A: How federated learning works, the adapter, the training algorithm, experimental comparisons, privacy mechanisms and reproducibility.
- Part B: The camera-node pipeline, performance measurements, SQLite data store, server console, node console and end-to-end demo.

The goal is to understand the design decisions well enough to answer "why did you implement it this way?" and "what happens when this component runs?" questions.&#x20;

Aswin.pdf

Aswin.pdf



# Part A — Federated learning infrastructure

## 1. The 30-second explanation

Learn the meaning of this explanation before memorising the wording.

"In FedMPS, five simulated camera nodes collaborate to train a shared face-recognition adapter without pooling their training data. The ArcFace backbone remains frozen, and only a small adapter with 66,240 parameters is trained. Each node trains locally and sends a weight update of approximately 0.26 MB. The server aggregates the updates using FedAvg, while FedProx helps control differences between local training distributions. We run 30 rounds and select a checkpoint using held-out validation identities. Our experiments show that federated training matches centralised training, although the adapter does not measurably improve on the frozen backbone in this dataset."

Copy explanation

The final sentence matters. Present the privacy-preserving training pipeline as the contribution; don't claim that it improves recognition accuracy when the measured results do not show that.

## 2. What is federated, and what is not?

This is the foundation for the rest of Part A.

Frozen ArcFace backbone

## 43.6 million parameters

A pretrained ResNet-50 face-recognition model. It converts a face into a 512-dimensional embedding. It remains unchanged during federated training.

Trainable residual adapter

## 66,240 parameters

A small neural network that adjusts the embedding generated from a degraded face. This is the component trained collaboratively across the five nodes.

| Property                   | Backbone     | Adapter                                           |
| -------------------------- | ------------ | ------------------------------------------------- |
| Architecture               | ResNet-50    | Residual MLP                                      |
| Parameters                 | 43.6 million | 66,240                                            |
| Trained in this project?   | No           | Yes                                               |
| Approximate size           | 174.4 MB     | 0.26 MB                                           |
| Same weights across nodes? | Yes          | A shared global version is updated between rounds |



Aswin.pdf

FedMPS_Capstone_Thesis.docx



### Why freeze the backbone?

There are two main reasons.

First, a pretrained face-recognition network has already learned general facial representations from extensive training. A camera site in this project has only a few hundred local training identities, which is not a suitable basis for training a 43.6-million-parameter recognition backbone from scratch.

Second, federating a small adapter substantially reduces communication requirements. Instead of transmitting a large backbone update, the nodes exchange the adapter's small update.

The guide reports a reduction of approximately 658 times per update, and a reduction in the overall communication volume from approximately 52.3 GB to 79.5 MB per training run.&#x20;

Aswin.pdf



### Understand the adapter formula

The guide gives this formula:

\\[ a(x)=\operatorname{normalise}\left( x+W_2\operatorname{Dropout}\left( \operatorname{GELU}(\operatorname{LayerNorm}(W_1x)) \right)\right) \\]

Read it from inside out:

1. \\(x\\) is the original 512-dimensional face embedding.
2. \\(W_1\\) projects the embedding from 512 dimensions to 64.
3. LayerNorm normalises the intermediate activations.
4. GELU adds nonlinearity.
5. Dropout with probability 0.1 helps regularise training.
6. \\(W_2\\) projects the result back from 64 to 512 dimensions.
7. The result is added to the original embedding through a residual connection.
8. The final embedding is normalised to unit length.

Why is the residual connection important?

The adapter can make a correction while retaining the original embedding as its starting point. Its final layer is initialised to zero, so initially the adapter behaves like an identity function:

\\[ a(x)=x \\]

That means round 0 is equivalent to using the frozen backbone alone. Any subsequent change comes from training the adapter.&#x20;

FedMPS_Capstone_Thesis.docx



Likely viva question: Why use a bottleneck of 64 dimensions?

The smaller hidden dimension reduces the trainable parameter count and limits overfitting. The thesis describes experiments in which a larger adapter overfit the relatively small collection of training identities at the nodes.

## 3. What happens in one federated round?

This is the core algorithm you should be able to explain on a whiteboard.

Step 1 — Server distributes the global adapter

The current shared weights go to all five nodes.

Node 0–1

Train locally using their own images or prepared embeddings.

Node 2–4

Train locally using their own images or prepared embeddings.

Step 2 — Each node calculates its update

\\[ d_k=w_k-w_g \\]

The change in adapter weights is sent instead of the raw training data.

Step 3 — Optional privacy protection

Clip the update and add DP noise when configured; use pairwise masks for secure aggregation.

Step 4 — Server aggregates updates

The masks cancel in the aggregate, and the server updates the global adapter.

Step 5 — Validate and select

Score the global model on held-out validation identities and remember the best round.

Aswin.pdf

### FedAvg: how the server combines updates

The server uses data-size-weighted aggregation:

\\[ w_g^{\text{new}} = w_g+\sum\_{k=1}^{5}\frac{n_k}{n}\\,d_k \\]

Where:

- \\(w_g\\) is the current global adapter.
- \\(w_k\\) is the adapter after local training at node \\(k\\).
- \\(d_k=w_k-w_g\\) is that node's update.
- \\(n_k\\) is the node's training image count.
- \\(n=\sum_k n_k\\) is the total number of training images.

Why weight by image count?

Because the nodes have unequal amounts of data. Without weighting, a very small node could contribute as much to the global model as the largest node, regardless of their dataset sizes.

### Memorise the training configuration

| Setting                                    | Value             |
| ------------------------------------------ | ----------------- |
| Nodes                                      | 5                 |
| Federated rounds                           | 30                |
| Local training                             | 1 epoch per round |
| Optimiser                                  | AdamW             |
| Learning rate                              | \\(10^{-4}\\)     |
| Batch size                                 | 128               |
| Loss                                       | Symmetric InfoNCE |
| Temperature, \\(\tau\\)                    | 0.07              |
| FedProx coefficient, \\(\mu\\)             | 3.0               |
| Images processed across nodes in one round | 41,420            |
| Total local node-epochs over 30 rounds     | 150               |



Aswin.pdf

FedMPS_Capstone_Thesis.docx



One subtle point: 30 rounds means 30 local epochs per participating node, not 30 epochs for the entire system. With five nodes participating in every round, the training run totals 150 node-epochs.

## 4. InfoNCE loss — What is the adapter learning?

The objective is to bring the degraded face's embedding closer to the clean embedding of the same person, while separating it from embeddings of other people.

For each paired example:

- The degraded face is passed through the adapter.
- Its clean counterpart acts as the positive example.
- Images of other identities in the batch act as negative examples.

The loss uses similarity scores scaled by the temperature \\(\tau=0.07\\). Higher similarity to the correct clean face relative to the negatives reduces the contrastive loss.

The guide specifies a symmetric InfoNCE loss, meaning the paired comparisons are evaluated in both directions. Multiple examples of the same person are excluded from the negative denominator, so the model does not learn to push the same identity away from itself.&#x20;

Aswin.pdf

FedMPS_Capstone_Thesis.docx



How to explain it in the review:

"InfoNCE is a contrastive learning loss. It trains the adapter to bring a degraded face embedding closer to the corresponding clean embedding while distinguishing it from other identities. Because the negative examples come from the same node, they represent difficult comparisons under that node's camera conditions."

### What does the temperature do?

The temperature scales the similarity logits before the contrastive softmax. A smaller temperature makes the loss more sensitive to differences among similarity scores. The project uses 0.07; it is a training hyperparameter, not a face-matching threshold.

## 5. FedProx — Why add a second term to the loss?

Each node has a different camera profile. One may produce smaller, blurrier faces; another may produce larger, clearer faces. Their data distributions are therefore non-IID, meaning they are not identically distributed.

Local training can move each node's adapter in a different direction. This is often called client drift.

FedProx adds a penalty to keep the locally trained weights near the global weights received at the start of a round:

\\[ L=L\_{\text{InfoNCE}} +\frac{\mu}{2}\lVert w-w_g\rVert^2 \\]

Where \\(\mu\\) controls the strength of that penalty.

- When \\(\mu=0\\), this term disappears, giving the FedAvg-only training setup.
- A larger \\(\mu\\) penalises larger deviations from the global model.
- In the reported configuration, \\(\mu=3.0\\).

### The important result

The guide reports that the FedAvg and FedProx sweeps changed Rank-1 by less than 0.001 across the tested settings. The thesis also says that client drift was not the main bottleneck in this experiment, in part because early stopping kept the adapter close to its initial state.&#x20;

Aswin.pdf

FedMPS_Capstone_Thesis.docx



Likely question: Why set \\(\mu=3.0\\) if the sweep only went up to 1.0?

The guide's answer is that the measured results changed very little across the sweep, so this configuration did not change the overall conclusion. State that it is the chosen configuration and acknowledge that the reported sweep does not directly cover 3.0.

## 6. Early stopping — Why does the experiment retain the best round?

A small adapter can overfit the training identities. Its performance on new identities may peak early and then decline.

The system therefore evaluates the global adapter after every round on held-out validation identities and keeps the round with the highest validation Rank-1.

The reported validation curve gives:

- Frozen-backbone validation Rank-1: 0.9486.
- Best illustrated validation Rank-1: 0.9514 at round 1.
- The selected checkpoint in that run is round 1.

The validation identities are separate from both the training identities and the final test identities. This helps avoid selecting a model based on its final test performance.&#x20;

Aswin.pdf

FedMPS_Capstone_Thesis.docx



Viva question: Why not simply take the last round?

Because additional training is not guaranteed to improve generalisation. Early stopping selects the strongest checkpoint on validation identities instead of assuming that more training is always better.

Remember: the validation result is used to select the model. The final test set is used to report the selected model's performance.

## 7. The experimental comparisons — Which training method actually wins?

This section is probably the most important part of Part A for your viva.

Aswin.pdf labels this section "The four arms of the experiment", although its table lists five configurations. Know all five so that you can compare the training approaches correctly.&#x20;

Aswin.pdf



| Configuration                              | Rank-1 | Interpretation                               |
| ------------------------------------------ | ------ | -------------------------------------------- |
| Frozen backbone                            | 0.9474 | No adapter training                          |
| Local-only, naïve                          | 0.9286 | Each node trains independently for 30 epochs |
| Local-only with early stopping and FedProx | 0.9473 | Fairer local-training comparison             |
| Federated FedMPS                           | 0.9473 | Proposed collaborative approach              |
| Centralised                                | 0.9476 | Training data pooled together                |

### Why were two local-only results necessary?

The naïve local-only setup trained each node's adapter for 30 epochs without the same protective training choices used in FedMPS. That version performed worse, with Rank-1 of 0.9286.

The improved local-only baseline uses early stopping and FedProx. Its Rank-1 is 0.9473, matching the federated result.

This corrected an earlier claim that federated training beat local-only training by 1.87 percentage points. That gap came from comparing federated training with the naïve, overfitted local baseline, not from collaboration alone.&#x20;

Aswin.pdf

FedMPS_Capstone_Thesis.docx



### How to explain the table

"The purpose of the comparison is to separate the benefit of collaboration from the effect of training choices. Naïve local training overfits and achieves 92.86% Rank-1. Once the local-only baseline uses early stopping and FedProx, it reaches 94.73%, matching the federated result. Centralised training reaches 94.76%. Therefore, the key finding is that federated training achieves comparable performance without pooling the training data."

### Why is the centralised result important?

Centralised training acts as a comparison point for what happens when the participating nodes' training data is pooled. If federated training performs comparably, it suggests that the distributed training approach has not sacrificed measurable identification performance in this experiment.

However, the centralised result is only an experimental comparison. It does not mean the server can safely collect the actual face data in a privacy-sensitive deployment.

The result to remember: the frozen backbone, fairly trained local adapter, federated adapter and centralised adapter all have Rank-1 values around 0.947–0.948. The adapter's contribution is too small to establish a measurable gain on this dataset.

## 8. Secure aggregation and differential privacy

These are two different privacy mechanisms. Don't describe them as though they do the same thing.

### 8.1 Secure aggregation

The server needs the aggregate of the five nodes' updates. Ideally, it should not see each node's individual update.

FedMPS implements pairwise masks using Diffie–Hellman shared secrets.

Each node has an update

The node's own model-weight change.

Apply pairwise masks

One node adds a pair's mask, while the other subtracts it.

Server sums the masked updates

Matching masks cancel, leaving the intended aggregate.

The protocol works as follows:

1. Each pair of nodes establishes a shared secret through Diffie–Hellman key exchange.
2. The shared secret, round number and node indices determine the mask seed.
3. One node adds the pairwise mask to its update, and the other subtracts it.
4. When the server sums the participating nodes' updates, the masks cancel.

The server can then aggregate the updates without receiving the individual unmasked updates, assuming the protocol executes as intended.&#x20;

Aswin.pdf

FedMPS_Capstone_Thesis.docx



Measured result: the guide reports Rank-1 of 0.9376 in both the masked and unmasked secure-aggregation comparison, with approximately 0.8% additional runtime.

Notice that these values are from the secure-aggregation experiment. Do not substitute them for the headline training-comparison results from the previous section.

### What if a node goes offline?

This is a useful limitation to know.

In the reported experiments, all five nodes participate in every round. If a node drops out halfway through a masked round, some masks may not cancel. The guide identifies Shamir-based self-mask recovery as future work.

A production-grade secure-aggregation protocol needs to handle client dropout correctly; ordinary pairwise mask cancellation alone does not solve that problem.

### 8.2 Differential privacy

Differential privacy works differently. It introduces calibrated randomness into model updates to limit how much the released result can reveal about an individual's contribution.

The guide reports this process:

1. Clip the update to an L2 norm bound of \\(C=0.128\\).
2. Add Gaussian noise.
3. Aggregate the protected updates.

The clipping bound limits the magnitude of an update. The noise multiplier determines how much noise is added relative to that bound.

The thesis evaluates a range of noise settings, but the project did not find a setting that provided useful privacy while retaining acceptable accuracy.&#x20;

FedMPS_Capstone_Thesis.docx



### Secure aggregation versus differential privacy

| Property                | Secure aggregation                                     | Differential privacy                               |
| ----------------------- | ------------------------------------------------------ | -------------------------------------------------- |
| Main mechanism          | Pairwise masks that cancel in a sum                    | Clipping and calibrated random noise               |
| Main aim                | Hide individual updates from the aggregator            | Limit information leakage through released updates |
| Aggregate usable?       | Yes, the masks cancel                                  | Yes, but noise affects the result                  |
| Finding in this project | No measured accuracy change in the reported comparison | No usable privacy–accuracy operating point found   |

Likely viva question: Does federated learning itself guarantee privacy?

"No. Federated learning keeps raw training data at the nodes, but model updates can themselves reveal information. Secure aggregation helps hide individual updates, while differential privacy attempts to bound information leakage through the training process. These protections have different assumptions and limitations."

The guide also reports a membership-inference attack AUC of 0.496, approximately chance-level performance for that particular attack. This is evidence about the tested attack, not proof that every possible privacy attack will fail.&#x20;

Aswin.pdf



## 9. Where did the experiments run, and how reproducible are they?

Aswin.pdf describes two parts of the experimental workflow.

The expensive embedding preparation ran on an AWS `g4dn.2xlarge` instance equipped with an NVIDIA T4 GPU. Preparing the data took approximately 38 minutes, and the experiment suite took approximately 22 minutes.

Federated training itself runs on the saved embeddings, which allows the training experiments to run on a laptop CPU. The five nodes, however, are simulated within a single process on one machine. This is not the same as five physically independent camera installations communicating over a real network.&#x20;

Aswin.pdf



### The reproducibility question

The guide records the following:

- AWS run: selected round 26, with test Rank-1 of 0.9448.
- Laptop run: selected round 1, with test Rank-1 of 0.9473.
- The measured validation differences between machines changed by at most 11 out of 2,590 probes.
- The selection of the winning round was decided by only three probes in the reported comparison.

These results show that small numerical differences can change which round wins when validation scores are extremely close. They also reinforce that the adapter's effect is small relative to the overall recognition performance.&#x20;

Aswin.pdf



### What to say

"We prepared the embeddings on an AWS T4 GPU, then ran the federated training experiments on the saved embeddings. This made it possible to repeat the training comparisons on a laptop. Small numerical differences changed the best validation round, demonstrating that the adapter's measured effect is marginal. Our five-node training implementation is still a simulation; deployment over a real network, for example with a framework such as Flower, is future work."

Likely question: Why don't all machines select the same round?

The validation scores are close enough that small floating-point differences can change the ordering of the rounds. When the difference between candidate rounds is determined by only a few probes, a small numerical variation can change the winner.

Likely question: Is Flower already used in your implementation?

According to the guide, Flower is an example of a possible future distributed deployment. Do not claim that the project already runs its nodes on Flower.

# Part B — Edge systems and dashboards

## 10. What does "edge processing" mean in FedMPS?

The edge is the camera site or local processing environment.

Rather than sending the entire camera stream to a central recognition server, each node processes its own video. The pipeline performs detection, tracking, quality checks, embedding extraction, matching and track-level voting locally.

Only a candidate alert enters that site's review queue. After a human confirms it, the selected crop and associated evidence are sent upstream.

This is both a software architecture decision and a privacy decision.&#x20;

Aswin.pdf

FedMPS_Capstone_Thesis.docx



## 11. The edge pipeline — Components, thresholds and performance

| Component      | What it does                                              | Main implementation                        |
| -------------- | --------------------------------------------------------- | ------------------------------------------ |
| Video scanning | Samples video at up to 8 fps and creates annotated output | `application/run_video_scan.py`            |
| Face detection | Detects faces and landmarks using SCRFD `det_10g`         | `fedmps/pipeline/detect.py`                |
| Tracking       | Connects face detections across frames                    | ByteTrack                                  |
| Quality gate   | Rejects low-quality faces                                 | `fedmps/pipeline/quality.py`               |
| Matching       | Compares face embeddings or protected templates           | `fedmps/pipeline/match.py`                 |
| Voting         | Requires agreement across several frames                  | `TrackVoter` in `fedmps/pipeline/match.py` |

Aswin.pdf

### The thresholds

| Parameter                   | Value                  |
| --------------------------- | ---------------------- |
| Maximum sampling rate       | 8 fps                  |
| Minimum eye separation      | 24 px                  |
| Mean brightness range       | 25–235                 |
| Maximum yaw                 | 45°                    |
| Cosine matching threshold   | 0.24                   |
| Template matching threshold | 0.2656                 |
| Voting rule                 | 3 of the last 5 frames |

These are the defaults described in the guide. The two matching modes use different score representations, so their thresholds are not interchangeable.&#x20;

Aswin.pdf



### Why do we need ByteTrack?

The detector tells us where faces appear in individual frames. Tracking associates detections across successive frames so that the system can treat multiple observations of one person as a track.

Without tracking, several matching frames of the same person could generate multiple independent alerts. With tracking and voting, the system can combine evidence over time and avoid raising an alert on every individual frame hit.

### What is the quality gate doing?

It rejects face observations that don't meet the minimum size, sharpness, brightness or pose requirements.

The idea is that attempting recognition on an extremely small or blurred face can create an unreliable result. Rather than allowing that observation to trigger an alert, the pipeline rejects it.

### Measured performance

Laptop CPU

AMD Ryzen 5 7520U

# 1–2.3 fps

Approximate measured speed

AWS GPU

NVIDIA T4 · ChokePoint video

# \~42 fps

14,751 frames in 5.9 minutes

Aswin.pdf

How to explain the performance result:

"The laptop CPU is adequate for scanning recorded footage after a missing-person report, which is the demonstrated use case. Continuous live monitoring at a busy camera site would require suitable GPU hardware or an edge accelerator. The measured GPU speed is hardware- and workload-dependent; it is not a guarantee for every deployment."

Likely viva question: Is the pipeline real-time?

The laptop implementation processes approximately 1–2.3 fps, which does not support assuming real-time processing for a full-rate camera stream. The T4 GPU benchmark is about 42 fps on the tested ChokePoint workload. Real deployments must be sized and evaluated on their actual camera and processing hardware.

## 12. SQLite — How the two consoles share operational data

The guide identifies `fedmps/store.py` as the module responsible for the shared SQLite store.

There are four tables:

`cases`

Stores missing-person cases, their status and expiry information.

`templates`

Stores node-specific keyed templates and related key fingerprints.

`alerts`

Stores candidate alerts and their review status.

`audit`

Records operational actions for traceability and oversight.

Aswin.pdf

### The store functions you should recognise

| Function               | Purpose                                                   |
| ---------------------- | --------------------------------------------------------- |
| `create_case`          | Opens a case, which expires after 30 days by default      |
| `issue_template`       | Issues a keyed 64-byte template to a node                 |
| `add_alert`            | Records a pending candidate alert                         |
| `review_alert`         | Records the operator's Confirm or Reject decision         |
| `reference_thumb`      | Opens the reference thumbnail and logs the view           |
| `close_case`           | Closes a case, revokes its templates and logs closure     |
| `audit` / `list_audit` | Records or retrieves audit entries                        |
| `precision_report`     | Calculates the share of reviewed alerts that were correct |

### The most important architectural limitation

Both consoles share one local SQLite database in the demo. This is useful for demonstrating the end-to-end workflow, but it is not the same as deploying independent databases and authenticated network APIs across five real camera sites.

A production system would need a different deployment architecture, suitable database services, authenticated communication, access controls and appropriate operational security. The guide explicitly identifies the single-store design as a demo limitation.&#x20;

Aswin.pdf



## 13. Server console — `application/server_app.py`

Default port: 8501

The server console is the investigator-facing part of the application.

It has four main tabs:

- Open a case
- Cases
- Confirmed sightings
- Audit log

The investigator supplies the case name, case reference, notes, expiry and reference photograph. The server embeds the photograph in memory and issues a differently keyed template to each of the five nodes.

The guide says the full reference photo is not stored; only a 200 × 200 thumbnail is retained for human review. The server dashboard shows confirmed sightings, not the nodes' pending alerts.&#x20;

Aswin.pdf

FedMPS_Capstone_Thesis.docx



### What happens when a case is opened?

1. The investigator enters the case details and selects the reference photo.
2. The server computes the face embedding.
3. The application produces node-specific keyed templates.
4. It stores the case and associated operational metadata.
5. The nodes use the active templates when processing their own footage.

### What does the server see?

According to the guide, the server sees confirmed sightings and their evidence, such as the candidate crop, score, vote information and operator. It does not see a pending alert before the node operator confirms it.

The dashboard's precision metric represents the share of reviewed alerts that were real:

\\[ \text{Precision} =\frac{\text{True positive alerts}} {\text{All reviewed alerts}} \\]

Likely viva question: Why not just send all candidates to the server and decide there?

Because that would centralise candidate faces before the local site has reviewed them. Requiring human confirmation at the node reduces unnecessary disclosure of bystanders' images and associates the disclosure with an accountable operator.

## 14. Node console — `application/node_app.py`

Default port: 8502

The node console is the operator-facing part of the application.

It provides the local video-scanning and candidate-review workflow. The review queue presents the missing-person reference thumbnail beside the candidate face, along with the score and the track's voting evidence.

### Understand the different tabs

Review queue: the operator examines pending alerts and confirms or rejects them.

Watchlist: the node sees case references and key fingerprints, not a list of reference photographs.

Training labels: confirmations are treated as positive examples; rejections can serve as hard negatives from that camera. Feeding these new labels into federated training is future work, not a completed automatic learning loop.

### An important privacy nuance

The statement that nodes do not receive the missing person's full photo does not mean the reviewer can never see a reference image. The system keeps keyed templates for matching, and a thumbnail can be displayed for the specific pending alert so the operator can make a decision. Viewing that thumbnail is logged.&#x20;

Aswin.pdf

FedMPS_Capstone_Thesis.docx



### Why is the node console so important?

It acts as:

1. False-positive control: an operator can reject an incorrect candidate.
2. An accountability mechanism: the decision is associated with an operator.
3. A source of labels: confirmed and rejected cases can provide supervised examples for potential future training.

Likely viva question: What if the operator makes the wrong decision?

A wrong confirmation can still happen; human review does not guarantee correctness. The system records who confirmed the sighting, keeps an audit trail and calculates precision over reviewed alerts. The guide also describes the labels as useful for future improvement, but it does not claim that the current system automatically fixes each mistaken decision.

## 15. The audit log — Following a case from creation to closure

The audit log complements the two consoles. It captures events such as:

- `case.create`
- `template.issue` for each of the five nodes
- `alert.raise`
- `reference.view`
- `alert.confirmed` or `alert.rejected`
- Case closure

Every entry identifies the action and actor, along with its timestamp. Both server and node operators can inspect the audit trail.&#x20;

Aswin.pdf



A typical demonstration would show the server opening a case, issuing five templates, Node 2 raising an alert, the operator viewing the reference and then confirming the candidate.

Likely question: What happens when a case expires?

Expired or closed cases stop appearing in the active watchlist returned to the nodes. Closing a case also revokes its templates and records the closure event.

Precision when describing this design: the application maintains an append-only audit trail by design, as described in the guide. Do not equate that alone with a cryptographically tamper-proof or production-hardened audit infrastructure.

# 16. Run the demo and explain what happens

Aswin.pdf includes a precise demonstration sequence. You should understand it as a complete workflow, not merely two commands.

## Step 1 — Start the server console

```
python -m streamlit run application/server_app.py --server.port 8501
```

## Step 2 — Start the node console

In another terminal:

```
python -m streamlit run application/node_app.py --server.port 8502
```

## Step 3 — Walk through the case

1. In the server console, open a case using `media/person-1.jpg`.
2. Open the Node 2 console and click Scan video.
3. Examine the pending alert in the node review queue.
4. Confirm the alert.
5. Return to the server console to see the confirmed sighting and inspect the audit log.

This sequence demonstrates case creation, template issuance, local video processing, alert generation, human review, confirmation and auditing.&#x20;

Aswin.pdf



### What are the demo results?

| Measurement                                   | Reported value    |
| --------------------------------------------- | ----------------- |
| Faces detected                                | 220               |
| Faces passed through the quality gate         | 40                |
| Faces rejected, primarily for being too small | Approximately 82% |
| Frame-level hits converted into an alert      | 40 hits → 1 alert |
| Example cosine-mode alert score               | 0.55              |
| Example template-mode alert score             | 0.387             |

The template-mode and cosine-mode scores use different representations, so the example values are not directly comparable. The demo illustrates the importance of filtering poor-quality faces and aggregating frame-level evidence into a track-level alert.&#x20;

Aswin.pdf



### Be careful with the reset command

The guide provides this command:

```
python application/demo_reset.py
```

It clears demo cases, templates, alerts, audit entries and scan outputs. The guide recommends backing up `artifacts/fedmps.sqlite` first if you need to retain the data.

# 17. Likely faculty questions: Part B

These are worth practising aloud.

Q1. Why did you choose Streamlit and SQLite?

"Streamlit let us implement the two demonstration consoles quickly, while SQLite was sufficient for the shared prototype database. The project focuses on the federated protocol, privacy mechanisms and review workflow. A production deployment would need authenticated APIs and a suitable server-side or distributed database architecture."

Q2. Can the server see a pending alert before the operator reviews it?

"No. Pending alerts remain in the local node's review queue. The server console shows confirmed sightings after the operator confirms them."

Q3. Why doesn't the node receive the missing person's full photograph?

"The node uses its case-specific keyed watchlist representation for matching. The full reference photograph is not distributed to every node. A thumbnail can be shown for a particular pending alert so the human reviewer can compare the candidate, and that access is recorded."

Q4. What is the difference between a frame-level hit and an alert?

"A frame-level hit is a positive matching result for one frame. An alert is generated at the track level after the required voting evidence is met. Several frame hits from the same track can therefore become one alert."

Q5. How does the system reduce false positives?

"It combines a quality gate, matching thresholds, track-level voting and human review. The quality gate rejects unreliable faces, voting requires repeated evidence, and an operator reviews the pending candidate before it is sent upstream."

Q6. Is federated learning deployed across five physical camera sites?

"No. The five-node experiment is simulated within one process on one machine. A true multi-machine network deployment is future work."

Q7. Can the current dashboard support continuous live CCTV monitoring?

"The measured laptop CPU speed is around 1–2.3 fps, while the tested T4 GPU workload achieves approximately 42 fps. Continuous monitoring requires appropriate hardware at each camera site, along with evaluation under the actual deployment conditions."

Q8. What happens when a case closes?

"The case is closed, its templates are revoked, the closure is logged, and the case is excluded from the nodes' active watchlists."

Q9. What happens to confirmations and rejections after human review?

"They are recorded as review decisions, contribute to operational precision reporting and can act as positive examples or hard negatives. Feeding these labels back into federated training remains future work."

Q10. What is the main limitation of the dashboard implementation?

"Both consoles currently share a local SQLite database. The prototype demonstrates the workflow, but it does not yet implement separate databases and secure network communication for independently operated camera sites."



Aswin.pdf

FedMPS_Capstone_Thesis.docx



# 18. Final revision sheet — the numbers and facts you must know

Use this table as your last-minute revision checklist.

| Topic                                      | Remember                                                            |
| ------------------------------------------ | ------------------------------------------------------------------- |
| Number of nodes                            | 5                                                                   |
| Frozen backbone                            | ArcFace ResNet-50, 43.6M parameters                                 |
| Trainable adapter                          | 66,240 parameters, approximately 0.26 MB                            |
| Adapter architecture                       | 512 → 64 → 512                                                      |
| Communication reduction                    | Approximately 658× per update                                       |
| Federated training                         | 30 rounds, 1 local epoch per round                                  |
| Optimiser and batch                        | AdamW, learning rate \\(10^{-4}\\), batch 128                       |
| Loss                                       | Symmetric InfoNCE, temperature 0.07                                 |
| FedProx                                    | \\(\mu=3.0\\)                                                       |
| Aggregation                                | Data-size-weighted FedAvg                                           |
| Best round in illustrated validation curve | Round 1                                                             |
| Main test Rank-1                           | Frozen 0.9474; federated 0.9473; centralised 0.9476                 |
| Secure aggregation comparison              | Same reported Rank-1 of 0.9376, approximately 0.8% runtime overhead |
| Differential privacy                       | Clip norm 0.128; no usable privacy–accuracy setting found           |
| Edge pipeline                              | SCRFD → ByteTrack → quality gate → embedding → match → vote         |
| Voting rule                                | 3 of the last 5 frames                                              |
| Default match thresholds                   | Cosine 0.24; template 0.2656                                        |
| Laptop performance                         | Approximately 1–2.3 fps                                             |
| T4 performance                             | Approximately 42 fps on the reported workload                       |
| SQLite tables                              | Cases, templates, alerts, audit                                     |
| Console ports                              | Server 8501; node 8502                                              |

Source: Aswin.pdf and the capstone thesis.&#x20;

Aswin.pdf

Aswin.pdf

FedMPS_Capstone_Thesis.docx



## The five points to communicate confidently

First, the backbone is frozen and the adapter is federated. This is what keeps each update small.

Second, FedAvg and FedProx have different roles. FedAvg aggregates updates; FedProx penalises excessive deviation from the global model during local training.

Third, federated learning is not the same as complete privacy. Secure aggregation, differential privacy and cancelable templates address different risks, and their limitations matter.

Fourth, edge processing and human review are core parts of the system. A model match is not an automatic disclosure; the local operator reviews and confirms a candidate.

Fifth, the results must be stated honestly. Federated training matches the centralised baseline, but neither the adapter nor pooling the data produces a measurable improvement over the frozen backbone in the main experiment. The system is an integrated prototype with a simulated federated deployment.

With these concepts, you have covered both halves of Aswin.pdf: the federated training infrastructure and the operational edge application. The most useful next exercise is a mock viva in which you explain a single round of training, then defend the privacy mechanisms and the design of the two consoles under follow-up questions.

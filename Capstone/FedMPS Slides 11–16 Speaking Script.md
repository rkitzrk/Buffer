# FedMPS Slides 11–16 Speaking Script

Nov 19, 2026 · @RK

## Slide 11: Datasets and input-output

**Cue:** two datasets, split by people, photo + video in, pending alert out.

"Now to implementation and results. I'll start with the data and the input and output of the system.

We used two datasets. The first is VGGFace2-HQ: 628,699 face images of 4,605 people. We split it by identity. Seventy percent of the people were spread across our five camera nodes for training, and thirty percent were held out for testing. So no test person is ever seen in training.

Since VGGFace2 is clean photographs, we degraded it to look like CCTV: blur, compression, noise and lighting changes. Each node has its own camera profile, so the nodes see different quality.

The second dataset is ChokePoint, real surveillance video: 29 people across 24 sequences.

The input to the system is a reference photo of the missing person and CCTV video. The system detects and tracks faces and compares them with the watchlist. The output is a pending alert with the face crop and a score. On the right, the enrolled person is boxed in red and everyone else in green.

The key point: the system raises a candidate alert. A human operator decides."

## Slide 12: Parameters

**Cue:** adapter, loss, FedProx, schedule, thresholds, privacy settings. Don't read the table; explain three things: the adapter, FedProx, the 3-of-5 vote.

"These are the settings behind the results.

The adapter is a small residual network: 512 inputs, a 256-unit hidden layer, 512 outputs, 263,424 parameters. It takes the ArcFace embedding of a degraded face and nudges it back toward where the clean face would be. ArcFace itself stays frozen.

We train it with a symmetric InfoNCE loss, temperature 0.07, which pulls a degraded face toward its own clean embedding and pushes it away from other people.

Federated training uses FedProx with mu 0.3. Each camera has different image quality, so local models drift apart. The proximal term holds each node close to the global model. We run 200 rounds over five nodes, using 15 percent of each node's data per round, with AdamW at learning rate 0.0001 and a cosine schedule. These settings were fixed in advance by a pilot on training identities only, so nothing was tuned on the test set.

For matching, the cosine threshold is 0.24, and 0.2656 for template matching. An alert needs 3 of the last 5 frames of the same tracked person to agree, so one lucky frame can't trigger it.

For privacy, the DP clipping norm is 0.313, set from the observed update sizes, and templates are 512 bits, which is 64 bytes."

## Slide 13: Experimental outcomes

**Cue:** four regimes, same test set. Federated equals pooling. Training alone is worse. Gain over frozen is small. Real CCTV, then privacy costs.

"We compared four setups on the same test set: the frozen ArcFace backbone, centralised training on pooled data, federated training, and local-only training at each node.

We report three metrics. Rank-1 is closed-set: is the right person the top match. TPIR at 1% FPIR is open-set: of the probes showing watchlist people, how many are correctly identified at the threshold where only 1% of non-watchlist probes raise a false alarm. EER is the point where false accepts and false rejects are equal; lower is better.

Federated reached rank-1 of 97.34 percent against 97.33 for centralised. On TPIR it is 91.07 against 90.80. On EER, centralised is marginally better, 11.7 against 11.8 percent. So federated is comparable to pooling, and the data never left the sites.

Local-only is clearly worse: rank-1 96.20 and TPIR 88.12. Training alone loses accuracy.

In fairness, the gain over the frozen backbone is small, TPIR 90.54 to 91.07. ArcFace does most of the work. Our finding is that federating costs nothing against pooling and avoids the loss from training alone.

On real CCTV, ChokePoint, the system alerted on 95.2 percent of enrolled walk-throughs, with zero wrong-name alerts. About 4 percent of crowd walk-throughs also raised an alert, which is why a human confirms every one.

On privacy: secure aggregation cost zero accuracy. The membership-inference attack scored 0.501, which is chance. Cancelable templates cost 4.7 rank-1 points. Differential privacy had no usable setting with only five sites.

On the right, the comparison with published work is indicative only. Those papers test clean images; our probes are 20 to 48 pixel CCTV faces."

**Avoid saying** that federated beats centralised on every metric. It is slightly ahead on rank-1 and TPIR, and slightly behind on EER.

## Slides 14 to 16: Summary, bibliography, close

**Slide 14 cue:** two findings, three limits, four next steps.

"To summarise. First, federated training matched centralised training: rank-1 of 0.9734 against 0.9733. Camera sites can train together without pooling data. Second, training alone at one site lost accuracy, so the design is to keep ArcFace frozen and federate a small shared adapter. The frozen backbone does most of the work.

The limitations: our degradation is simulated, and the real-CCTV test covers only 29 people. We used one random seed. And differential privacy gave no usable setting with five sites.

Next steps are harder real footage, per-site thresholds, several seeds, and secure aggregation that tolerates nodes dropping out."

**Slide 15 cue:** one line on the foundation; name three references.

"These are the papers and datasets behind the project. The three that matter most for the method are \[15\] ArcFace for the recognition backbone, \[16\] FedAvg for aggregation, and \[17\] FedProx for non-IID training. VGGFace2-HQ and ChokePoint are our datasets. The full list of 27 is in the thesis."

**Slide 16 cue:** thank, eye contact, invite questions.

"That concludes our presentation. FedMPS shows that computer vision, federated learning and privacy techniques can work together for missing-person search across camera sites, with a human confirming every alert. Thank you. We're happy to take questions."

## Likely questions and short answers

| Question | Answer |
| --- | --- |
| Why VGGFace2-HQ? | It has many identities and images per person (4,605 people, 628,699 images), enough to split by identity across five nodes and still hold out 30% of people. |
| Why also ChokePoint? | VGGFace2 degradation is simulated. ChokePoint is real surveillance video, so it checks the system outside our own degradation model. |
| Rank-1 vs TPIR vs EER? | Rank-1: right person is the top match (closed-set). TPIR at 1% FPIR: share of watchlist probes identified when only 1% of non-watchlist probes false-alarm (open-set). EER: where false accepts equal false rejects; lower is better. |
| Why 512 to 256 to 512? | A small residual MLP that starts as an identity map, so round 0 equals the frozen model. The 256 width was fixed by a pilot on training identities only. Check `results/pilot_vgghq.json` for the exact comparison before you answer in detail. |
| What is FedProx mu = 0.3? | A penalty that keeps each node's weights near the global model, which limits drift between nodes with different camera quality. Mu 0.3 gave the best TPIR at 1% FPIR in the pilot. |
| Why 200 rounds? | At 15% of each node's data per round, one round is a small step. The convergence figure shows a small, stable gain that plateaus. |
| Why is local-only worse? | One site's data is too small and too narrow for the adapter, so it overfits. Every single-site adapter fell below the frozen backbone. |
| What does 3-of-5 voting mean? | An alert needs 3 of the last 5 frames of the same tracked person to match. It stops single noisy frames from raising alerts. |
| Why did DP fail? | With only five nodes, the noise needed for a meaningful privacy budget swamps the signal. Even the weakest noise gave epsilon about 252, and accuracy collapsed. |
| The adapter barely beats frozen. Why bother? | Federated equals pooling, so privacy costs no accuracy, and local training alone is worse. The frozen backbone does most of the work, and we say so. |
| Is 95.2% real-world accuracy? | No. It is 29 people on one dataset, with about a 4% crowd false-alarm rate. A human confirms every alert for that reason. |
| Why is the paper comparison lower on some rows? | Those papers test clean images. Our probes are 20 to 48 pixel CCTV faces. We match \[10\] (57.5% vs 57.1%) and are close on EER. |
| Limitations? | Simulated degradation, 29-person real test, one seed, no usable DP setting. Be direct about these; they are findings, not failures. |

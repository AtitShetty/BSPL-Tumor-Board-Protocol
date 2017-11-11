# **Tumor Board Protocol**


### 1.  Below is the protocol that represents all the interactions that can successfully help a patient with diagnosis of cancer.

Here the patientID is they key, that will be generated when a patient first visit physician. The protocol is completed when the physician gives patient the findings.

``` 
Diagnosis{

  roles Patient, Physician, Radiologist, Pathologist, Board // Patricia, Primo, Radia, Partho, TumorBoard
  parameters out patientID key, out findings

  private radiologistName, calcificationsReport, radiopID, recommendBiopsy, biopsyID, biopsyFindings, tissueSample, pathologistFindings,adjudicatedFindings

  // Patient Patricia visits physician Primo for normal checkup.
  // If Primo is suspicious, he recommends Patricia to visit a Radiologist, otherwise he gives Patricia his findins indicating good health.

  Patient -> Physician : VisitPhysician[out patientID]
  Physician -> Patient : NormalResult[in patientID, out findings]
  Physician -> Patient : RecommendRadiologist[in patientID, out radiologistName]


  // Patricia visits radiologist Radia for biopsy.
  // Radia send her findings to Primo, and recommends a biopsy if she has found calcifications.
  // Primo will either recommend biopsy to Patricia or not, depending on Radia's findings.

  Patient -> Radiologist : VisitRadiologist[in patientID, in radiologistName, out radiopID]
  Radiologist -> Physician : NoCalicifactions[in patientID, in radiopID, out calcificationsReport]
  Radiologist -> Physician : CalcificationsFound[in patientID, in radiopID, out calcificationsReport, out recommendBiopsy]
  Physician -> Patient : NormalRadiologistReport[in patientID, in calcificationsReport, out findings]
  Physician -> Patient : RecommendForBiopsy[in patientID, in calcificationsReport, in recommendBiopsy]


  // If biopsy is recommended, Patricia will visit Radia for biopsy.
  // Radia will perform biopsy and send her findings along with tissue sample to pathologist Partho.

  Patient -> Radiologist : VisitRadiologistForBiopsy[in patientID, out biopsyID]
  Radiologist -> Pathologist : AnalyzeSpecimen[in patientID, in biopsyID, out biopsyFindings, out tissueSample]

  // Partho on receiving tumor sample will perform analysis.
  // If his findings match with Radia's he will send his report to Primo, otherwise he will send his findings and Radia's findings to Tumor board.
  // If Primo receives findings from Partho, he will give the findings to Patricia.

  Pathologist -> Physician : AgreeRadiologistFindings[in patientID, in biopsyFindings, in tissueSample, out pathologistFindings]
  Physician -> Patient : PathologistFindings[in patientID, in pathologistFindings, out findings]
  Pathologist -> Board : DisagreeRadiologistFindings[in patientID, in biopsyFindings, in tissueSample, out pathologistFindings]

  // When Tumor board receives Partho's findings, they will compare it with Radia's.
  // They will send the adjudicated finding to Primo to forward it to Patricia.
  // They will also send the findings to both Radia and Partho.

  Board -> Physician : BoardFindings[in patientID, in biopsyFindings, in pathologistFindings, out adjudicatedFindings]
  Board -> Radiologist: AdjudicatedFindingsToRadiologist[in patientID, in adjudicatedFindings]
  Board -> Pathologist: AdjudicatedFindingsToPathologist[in patientID, in adjudicatedFindings]
  Physician -> Patient: FinalFindings[in patientID, in adjudicatedFindings, out findings]

}

```


### 2. Following are the two issues that arises in the above protocol:

 1. When Patricia visits Primo for a checkup, Primo can either recommend Patricia to visit radiologist or give her findings and tell her that she is alright.

 ```
{
  // Patient Patricia visits physician Primo for normal checkup.
  // If Primo is suspicious, he recommends Patricia to visit a Radiologist, otherwise he gives Patricia his findins indicating good health.

  Patient -> Physician : VisitPhysician[out patientID]
  Physician -> Patient : NormalResult[in patientID, out findings]
  Physician -> Patient : RecommendRadiologist[in patientID, out radiologistName]
}
 ```

 The issue with above protocol is that implies that Primo can tell Patricia that she is fine and also recommend her for a biopsy.

 The messages are not mutually exclusive, and thus not safe.


 2.  Another issue with the above protocol is breaking of causality in the scenario when Patricia has to visit a radiologist.

 ```
{
  // Patricia visits radiologist Radia for biopsy.
  // Radia send her findings to Primo, and recommends a biopsy if she has found calcifications.
  // Primo will either recommend biopsy to Patricia or not, depending on Radia's findings.

  Patient -> Radiologist : VisitRadiologist[in patientID, in radiologistName, out radiopID]
  Radiologist -> Physician : NoCalicifactions[in patientID, in radiopID, out calcificationsReport]
  Radiologist -> Physician : CalcificationsFound[in patientID, in radiopID, out calcificationsReport, out recommendBiopsy]
  Physician -> Patient : NormalRadiologistReport[in patientID, in calcificationsReport, out findings]
  Physician -> Patient : RecommendForBiopsy[in patientID, in calcificationsReport, in recommendBiopsy]


  // If biopsy is recommended, Patricia will visit Radia for biopsy.
  // Radia will perform biopsy and send her findings along with tissue sample to pathologist Partho.

  Patient -> Radiologist : VisitRadiologistForBiopsy[in patientID, out biopsyID]
  Radiologist -> Pathologist : AnalyzeSpecimen[in patientID, in biopsyID, out biopsyFindings, out tissueSample]
}
 ```

 We can see that Patricia visit to Radia for biopsy is not dependent on Primo recommending her. This will make a protocol as unsafe, as multiple *findings* can be enacted.


### 3. Below is the corrected version of above protocol

```
Diagnosis{

  roles Patient, Physician, Radiologist, Pathologist, Board // Patricia, Primo, Radia, Partho, TumorBoard
  parameters out patientID key, out findings

  private visitLogs, radiologistName, calcificationsReport, radiopID, recommendBiopsy,radiologistNameForBiopsy, biopsyID, biopsyFindings, tissueSample, pathologistFindings,adjudicatedFindings

  // Patient Patricia visits physician Primo for normal checkup.
  // If Primo is suspicious, he recommends Patricia to visit a Radiologist, otherwise he gives Patricia his findins indicating good health.

  Patient -> Physician : VisitPhysician[out patientID]
  Physician -> Patient : NormalResult[in patientID, out findings, out visitLogs]
  Physician -> Patient : RecommendRadiologist[in patientID, out radiologistName, out visitLogs]

  // Patricia visits radiologist Radia for biopsy.
  // Radia send her findings to Primo, and recommends a biopsy if she has found calcifications.
  // Primo will either recommend biopsy to Patricia or not, depending on Radia's findings.

  Patient -> Radiologist : VisitRadiologist[in patientID, in radiologistName, out radiopID]
  Radiologist -> Physician : NoCalicifactions[in patientID, in radiopID, out calcificationsReport]
  Radiologist -> Physician : CalcificationFound[in patientID, in radiopID, out calcificationsReport, out recommendBiopsy]
  Physician -> Patient : NormalRadiologistReport[in patientID, in calcificationsReport, out findings]
  Physician -> Patient : RecommendForBiopsy[in patientID, in calcificationsReport, in recommendBiopsy, out radiologistNameForBiopsy]

  // If biopsy is recommended, Patricia will visit Radia for biopsy.
  // Radia will perform biopsy and send her findings along with tissue sample to pathologist Partho.

  Patient -> Radiologist : VisitRadiologistForBiopsy[in patientID, in radiologistNameForBiopsy, out biopsyID]
  Radiologist -> Pathologist : AnalyzeSpecimen[in patientID, in biopsyID, out biopsyFindings, out tissueSample]

  // Partho on receiving tumor sample will perform analysis.
  // If his findings match with Radia's he will send his report to Primo, otherwise he will send his findings and Radia's findings to Tumor board.
  // If Primo receives findings from Partho, he will give the findings to Patricia.

  Pathologist -> Physician : AgreeRadiologistFindings[in patientID, in biopsyFindings, in tissueSample, out pathologistFindings]
  Physician -> Patient : PathologistFindings[in patientID, in pathologistFindings, out findings]
  Pathologist -> Board : DisagreeRadiologistFindings[in patientID, in biopsyFindings, in tissueSample, out pathologistFindings]

  // When Tumor board receives Partho's findings, they will compare it with Radia's.
  // They will send the adjudicated finding to Primo to forward it to Patricia.
  // They will also send the findings to both Radia and Partho.

  Board -> Physician : BoardFindings[in patientID, in biopsyFindings, in pathologistFindings, out adjudicatedFindings]
  Board -> Radiologist: AdjudicatedFindingsToRadiologist[in patientID, in adjudicatedFindings]
  Board -> Pathologist: AdjudicatedFindingsToPathologist[in patientID, in adjudicatedFindings]
  Physician -> Patient: FinalFindings[in patientID, in adjudicatedFindings, out findings]

}

```

### 4. The above corrected protocol resolves the issues highlighted previously as follows:

- The first issue was resolved by adding a private parameter *visitLogs*. This will guarantee that both the messages are mutually exclusive.

- The second issue was resolved by using a private parameter *radiologistNameForBiopsy*. This is generated only when Primo recommends a biopsy for Patricia. Thus, Patricia will not visit Radia for biopsy of this is not generated and there will not be multiple enactments for *findings*.
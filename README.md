# dicom2ion-js
JavaScript implementation of a DICOM P10 converter to Ion

Project Status: pre-release software, do not use yet

## Why convert DICOM to Ion?
- ION supports two encodings - a human readable format like JSON and a compact binary format.  With other codecs, you either get one or the other
- ION is self describing - no external schema required.  Self describing codecs are not as space efficient as schema based codecs, but are easier to work with as they are self contained
- ION has a rich type system - can store binary data, high precision data, timestamps, annotations and symbolic expressions.  JSON has precision issues with Number types
  and poor support for binary (must encode in base64).  
- ION is optimized for reading/parsing - enables efficient sparse/shallow reads.  ION is much faster at decoding than JSON and competitive with other codecs (protobuf, etc)
- ION has libraries for most popular languages.  Protobuf and JSON have the largest language support, all others are lacking in several ways
- ION will be supported for a very long time - it used internally at Amazon
- ION has direct support for hashing.

Read more here:
https://amzn.github.io/ion-docs/guides/why.html


## Design Thoughts:

- Support async iterator as input so we can design for a full streaming implementation
- Make the attribute->value as simple and lightweight as possible (basically TAG=VALUE)
- Preserve the VRs in the original DICOM P10, but only the ones we don't already know (private vrs and multi-vr attributes)
- Use human friendly (Keyword) names for attributes (rather than group/element)
- Group related attributes together (e.g. patient, study, series, instance groups) to improve read/access/parse times (and human comprehension)
- Organize groups in the order of most frequently used (uids first, patient details next, etc)
- Must be possible to regenerate DICOM P10 from ION Format.  Ideally bit for bit lossless, but semantic equivalence is acceptable
- Store the sha256 digest of the original DICOM P10 so we can very integrity later
- Store the sha256 for each referenced data item so we can verify integrity later

### Input Parameters
- Stream to source DICOM P10
- Stream Info (optional)
  - uri to source DICOM P10 file
  - creation date
  - modification date
- Encoding Algorithm Parameters (optional)
  - privateAttributeMaxInlineLength - defaults to 256
  - standardAttributeMaxInlineLength - defaults to 256

### Returns async interable stream with ion data

### Output Schema
- Attribute Grouping
  - Enables faster parsing/lookups as groups can be skipped
  - Private attributes put in their own group
  - Multiple groups allow enable easier reading/comprehension of data (patient name not mixed with photometric interpretation)
- Uses human readable names vs tags for common attribute groups
  - Easier to read/debug
- Stores VRs separately from values
  - Easier to read/debug.  You rarely need the VR anyway
- Does not parse multi-valude string values into arrays
  - Easier to read/debug
- Encodes multi valued numeric types into binary data
  - these can be very large (e.g. LUTs) and rarely need to be human readable

### Example Output
```javascript
{
  sourceInfo: {
    uri: "file:///workspaces/dicom2ion-js/test/fixtures/CT0012.not_fragmented_bot_jpeg_ls.80.dcm"
  },
  options: {
    maximumInlineDataLength: {
      standard: 256,
      private: 256
    }
  },
  fileInfo: {
    sha256: "dc3ff8e550c833236bbee92d163762698b7b0b7b68a1af1b060243580741b7a6"
  },
  dataSet: {
    groups: {
      uidAttrs: {
        TransferSyntaxUID: "1.2.840.10008.1.2.4.80",
        InstanceCreatorUID: "1.3.6.1.4.1.5962.3",
        SOPClassUID: "1.2.840.10008.5.1.4.1.1.2.1",
        SOPInstanceUID: "1.3.6.1.4.1.5962.1.1.10.3.1.1166562673.14401",
        StudyInstanceUID: "1.3.6.1.4.1.5962.1.2.10.1166562673.14401",
        SeriesInstanceUID: "1.3.6.1.4.1.5962.1.3.10.3.1166562673.14401",
        FrameOfReferenceUID: "1.3.6.1.4.1.5962.1.4.10.1.1166562673.14401"
      },
      imageAttrs: {
        ImageType: "DERIVED\\PRIMARY\\PERFUSION\\RCBF",
        SamplesPerPixel: 1,
        PhotometricInterpretation: "MONOCHROME2",
        Rows: 512,
        Columns: 512,
        BitsAllocated: 16,
        BitsStored: 16,
        HighBit: 15,
        PixelRepresentation: 0
      },
      patientAttrs: {
        PatientName: "Perfusion^MCA Stroke",
        PatientID: "0010",
        PatientBirthDate: "19500704",
        PatientSex: "M"
      },
      studyAttrs: {
        StudyDate: "20061219",
        StudyTime: "111154.812",
        AccessionNumber: "0010",
        StudyDescription: null,
        StudyID: "0010"
      },
      seriesAttrs: {
        SeriesDate: "20061219",
        SeriesTime: "110929.984",
        Modality: "CT",
        SeriesDescription: null,
        SeriesNumber: "3"
      },
      instanceAttrs: {
        SpecificCharacterSet: "ISO_IR 100",
        InstanceCreationDate: "20061219",
        InstanceCreationTime: "202309",
        ContentDate: "20061219",
        ContentTime: "110930.671",
        AcquisitionNumber: "1",
        InstanceNumber: "1"
      },
      equipmentAttrs: {
        ImplementationVersionName: "OFFIS_DCMTK_361",
        SourceApplicationEntityTitle: "CLUNIE1",
        Manufacturer: "Acme Medical Devices",
        InstitutionName: "St. Nowhere Hospital",
        StationName: "CONSOLE01",
        ManufacturerModelName: "Super Dooper Scanner"
      }
    },
    standardAttrs: {
      FileMetaInformationGroupLength: {{0AAAAA==}},
      FileMetaInformationVersion: {{AAE=}},
      MediaStorageSOPClassUID: "1.2.840.10008.5.1.4.1.1.2.1",
      MediaStorageSOPInstanceUID: "1.3.6.1.4.1.5962.1.1.10.3.1.1166562673.14401",
      ImplementationClassUID: "1.2.276.0.7230010.3.0.3.6.1",
      ReferringPhysicianName: "Thomas^Albert",
      TimezoneOffsetFromUTC: "-0500",
      PerformingPhysicianName: "Smith^John",
      NameOfPhysiciansReadingStudy: "Smith^John",
      OperatorsName: "Jones^Molly",
      ReferencedRawDataSequence: [
        {
          standardAttrs: {
            ReferencedSeriesSequence: [
              {
                standardAttrs: {
                  ReferencedSOPSequence: [
                    {
                      standardAttrs: {
                        ReferencedSOPClassUID: "1.2.840.10008.5.1.4.1.1.66",
                        ReferencedSOPInstanceUID: "1.3.6.1.4.1.5962.1.9.10.1.1166562673.14401"
                      }
                    }
                  ],
                  SeriesInstanceUID: "1.3.6.1.4.1.5962.1.3.10.3.1166562673.14401"
                }
              }
            ],
            StudyInstanceUID: "1.3.6.1.4.1.5962.1.2.10.1166562673.14401"
          }
        }
      ],
      PixelPresentation: "COLOR",
      VolumetricProperties: "VOLUME",
      VolumeBasedCalculationTechnique: "NONE",
      PatientAge: "052Y",
      PatientSize: "1.6",
      PatientWeight: "75",
      ContrastBolusAgentSequence: [
        {
          standardAttrs: {
            CodeValue: "C-B0322",
            CodingSchemeDesignator: "SRT",
            CodeMeaning: "Iohexol",
            ContrastBolusAdministrationRouteSequence: [
              {
                standardAttrs: {
                  CodeValue: "G-D101",
                  CodingSchemeDesignator: "SNM3",
                  CodeMeaning: "Intravenous route"
                }
              }
            ],
            ContrastBolusVolume: "150",
            ContrastBolusIngredientConcentration: "300",
            ContrastBolusAgentNumber: 1,
            ContrastBolusIngredientCodeSequence: [
              {
                standardAttrs: {
                  CodeValue: "C-11400",
                  CodingSchemeDesignator: "SRT",
                  CodeMeaning: "Iodine"
                }
              }
            ]
          }
        }
      ],
      DeviceSerialNumber: "123456",
      SoftwareVersions: "1.00",
      PatientPosition: "HFS",
      ContentQualification: "PRODUCT",
      PositionReferenceIndicator: null,
      DimensionOrganizationSequence: [
        {
          standardAttrs: {
            DimensionOrganizationUID: "1.3.6.1.4.1.5962.1.6.10.3.0.1166562673.14401"
          }
        }
      ],
      DimensionIndexSequence: [
        {
          standardAttrs: {
            DimensionOrganizationUID: "1.3.6.1.4.1.5962.1.6.10.3.0.1166562673.14401",
            DimensionIndexPointer: "00209056",
            FunctionalGroupPointer: "00209111"
          }
        },
        {
          standardAttrs: {
            DimensionOrganizationUID: "1.3.6.1.4.1.5962.1.6.10.3.0.1166562673.14401",
            DimensionIndexPointer: "00209057",
            FunctionalGroupPointer: "00209111"
          }
        }
      ],
      NumberOfFrames: "2",
      BurnedInAnnotation: "NO",
      RedPaletteColorLookupTableDescriptor: {{ZAAABBAA}},
      GreenPaletteColorLookupTableDescriptor: {{ZAAABBAA}},
      BluePaletteColorLookupTableDescriptor: {{ZAAABBAA}},
      RedPaletteColorLookupTableData: {{AAEAAQABAAEAAQABAAEAAQABAAEAAQABAAEAAQABAAEAAQABAAEAAQABAAEAAQABAAEAAQABAAEAAQABAAEAAQABAAEAAQABAAEAAQABAAEAAQABAAEBAgIDAwRTB+MKcw4EEpQVJBm1HEUg1SNmJ/YqhS0WMaY0NzhUPHE/6EJ4RghKmU0CUR9USVfZWnZelWO0aNJtMnTge4+CPYrqkJiYR6D1p6OvUbf/vq3GW86i1bbcZOQS7MDzbfr+/j////////////8=}},
      GreenPaletteColorLookupTableData: {{AAEAAQABAAEAAQABAAEAAQABAAGAAQ8EBQZUCL4MUBGuFcsZ6h6VIycoRCxiMIE1njm9PttC+UcXTDZRVFVxWZBermLLZulqCHAldEN4Yn2AgJ+Fvordj/uUGpqHoDSn4a2btLu73MKWyUPQ8Nad3X7k9+qk8dz1+fkY//////////////////////////////////////////////////////////////////////////////////////////////////////8=}},
      BluePaletteColorLookupTableData: {{AAE9C3kUtx6VMENEvFc2a3wXN5Cjox23/sqd3bXo8/Mx/v////////////////////////////////////////////////////////////////////////////////////8L/3397/uR91Tuy+Qb22zSY8kmwF63rq5ypTac+ZLwiUGBRHkIcMtmj11TVBZL2kHEOBUwqCmJJGsfAx6SICIjWSeUL9E4DEGISTdSv1r9ZDtw+Xq4hPWONJpxpK+v7bnRxGjOpNc=}},
      LossyImageCompression: "00",
      AcquisitionContextSequence: [
      ],
      PresentationLUTShape: "IDENTITY",
      SharedFunctionalGroupsSequence: [
        {
          standardAttrs: {
            CTImageFrameTypeSequence: [
              {
                standardAttrs: {
                  FrameType: "DERIVED\\PRIMARY\\PERFUSION\\RCBF",
                  PixelPresentation: "COLOR",
                  VolumetricProperties: "VOLUME",
                  VolumeBasedCalculationTechnique: "NONE"
                }
              }
            ],
            ContrastBolusUsageSequence: [
              {
                standardAttrs: {
                  ContrastBolusAgentNumber: 1,
                  ContrastBolusAgentAdministered: "YES",
                  ContrastBolusAgentDetected: "YES",
                  ContrastBolusAgentPhase: "DYNAMIC"
                }
              }
            ],
            IrradiationEventIdentificationSequence: [
              {
                standardAttrs: {
                  IrradiationEventUID: "1.3.6.1.4.1.5962.1.10.10.3.1.1166562673.14401"
                }
              }
            ],
            FrameAnatomySequence: [
              {
                standardAttrs: {
                  AnatomicRegionSequence: [
                    {
                      standardAttrs: {
                        CodeValue: "T-A0100",
                        CodingSchemeDesignator: "SNM3",
                        CodeMeaning: "Brain"
                      }
                    }
                  ],
                  FrameLaterality: "U"
                }
              }
            ],
            PlaneOrientationSequence: [
              {
                standardAttrs: {
                  ImageOrientationPatient: "-1.00000\\0.00000\\0.00000\\0.00000\\1.00000\\0.00000"
                }
              }
            ],
            PixelMeasuresSequence: [
              {
                standardAttrs: {
                  SliceThickness: "10.0000",
                  PixelSpacing: "0.388672\\0.388672"
                }
              }
            ],
            FrameVOILUTSequence: [
              {
                standardAttrs: {
                  WindowCenter: "49.0000",
                  WindowWidth: "102.000"
                }
              }
            ],
            PixelValueTransformationSequence: [
              {
                standardAttrs: {
                  RescaleIntercept: "-1024.00",
                  RescaleSlope: "1.00000",
                  RescaleType: "US"
                }
              }
            ],
            RealWorldValueMappingSequence: [
              {
                standardAttrs: {
                  LUTExplanation: "Regional Cerebral Blood Flow",
                  MeasurementUnitsCodeSequence: [
                    {
                      standardAttrs: {
                        CodeValue: "ml/100ml/s",
                        CodingSchemeDesignator: "UCUM",
                        CodingSchemeVersion: "1.4",
                        CodeMeaning: "ml/100ml/s"
                      }
                    }
                  ],
                  LUTLabel: "RCBF",
                  RealWorldValueLastValueMapped: 4095,
                  RealWorldValueFirstValueMapped: 0,
                  RealWorldValueIntercept: -1024,
                  RealWorldValueSlope: 1
                }
              }
            ]
          }
        }
      ],
      PerFrameFunctionalGroupsSequence: [
        {
          standardAttrs: {
            FrameContentSequence: [
              {
                standardAttrs: {
                  StackID: "1",
                  InStackPositionNumber: {{AgAAAA==}},
                  FrameAcquisitionNumber: 1,
                  DimensionIndexValues: {{AQAAAAIAAAA=}}
                }
              }
            ],
            PlanePositionSequence: [
              {
                standardAttrs: {
                  ImagePositionPatient: "99.5000\\-301.500\\-159.000"
                }
              }
            ]
          }
        },
        {
          standardAttrs: {
            FrameContentSequence: [
              {
                standardAttrs: {
                  StackID: "1",
                  InStackPositionNumber: {{AQAAAA==}},
                  FrameAcquisitionNumber: 1,
                  DimensionIndexValues: {{AQAAAAEAAAA=}}
                }
              }
            ],
            PlanePositionSequence: [
              {
                standardAttrs: {
                  ImagePositionPatient: "99.5000\\-301.500\\-149.000"
                }
              }
            ]
          }
        }
      ],
      PixelData: {
        dataOffset: 3920,
        length: 85608,
        sha256: "fc003556dfa33c59e59a9575f2eb8ed4a7cb5349a7159d0abf85fd8ab7948a6b",
        vr: "OB",
        fragments: [
          {
            offset: 0,
            position: 3944,
            length: 46408
          },
          {
            offset: 46416,
            position: 50360,
            length: 39160
          }
        ],
        basicOffsetTable: [
          0,
          46416
        ],
        encapsulatedPixelData: true
      }
    }
  }
}
```
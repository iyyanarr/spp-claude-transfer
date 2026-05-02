# Detailed Manufacturing Process Flow

This diagram illustrates the comprehensive flow of logic, data, and material through the Shree Polymer Custom App.

```mermaid
graph TD
    %% Nodes
    subgraph Receiving_Process [Receiving & Mixing]
        DCR[Delivery Challan Receipt]
        DCR_Choice{Hold Receipt?}
        SE_Inward[Stock Entry: Material Transfer<br/>(To U3 Store)]
        SE_Hold[Stock Entry: Material Transfer<br/>(To Mixing Center)]
        WO_Mix[Work Order: Mixing]
        JC_Mix[Job Card: Mixing]
    end

    subgraph Quality_Control [Quality Assurance]
        CI[Compound Inspection]
        CI_Check{Inspection Status}
        SE_Approve[Stock Entry: Repack/Transfer<br/>(To Ready Store)]
        SE_Reject[Stock Entry: Material Transfer<br/>(To Rejection Warehouse)]
    end

    subgraph Blanking_Process [Blanking & Preparation]
        CBT[Cut Bit Transfer]
        SE_CBT[Stock Entry: Repack<br/>(To Cut Bit Warehouse)]
        
        BDE[Blanking DC Entry]
        BDE_Val[Validate Stock & Bom]
        SE_Blank[Stock Entry: Manufacture<br/>(Consume Sheet -> Produce Blank)]
        AM_Blank[Asset Movement: Bin]
        
        BBIE[Blank Bin Inward Entry]
        BBIE_Choice{Move to Cut Bit?}
        AM_Inward[Asset Movement: Receive Bin]
        SE_Inward_Repack[Stock Entry: Repack<br/>(To Cut Bit Warehouse)]
    end

    subgraph Production_Planning [Planning & Scheduling]
        WP[Add On Work Planning]
        WP_Gen[Generate Lot # & Barcodes]
        WO_Mould[Work Order: Moulding]
        JC_Mould[Job Card: Moulding]
    end

    subgraph Moulding_Execution [Moulding Operations]
        BBI[Blank Bin Issue]
        BBI_Val[Validate Bin vs BOM]
        
        MPE[Moulding Production Entry]
        MPE_Val{Validate Actual Stock}
        MPE_Entry[Record Production Output]
        SE_Mould[Stock Entry: Manufacture<br/>(Consume Blank -> Produce Part)]
        JC_Update[Update Job Card]
    end

    subgraph Post_Production [Finishing & Dispatch]
        DDE[Deflashing Despatch Entry]
        SE_Def_Out[Stock Entry: Material Transfer<br/>(To Deflashing Area)]
        
        RDE[Receive Deflashing Entry]
        SE_Def_In[Stock Entry: Material Transfer<br/>(From Deflashing Area)]
        
        LRT[Lot Resource Tagging]
        LRT_Pack[Packing / Final QC]
    end

    %% Edges - Receiving
    DCR --> DCR_Choice
    DCR_Choice -- No --> SE_Inward
    DCR_Choice -- Yes --> SE_Hold
    SE_Inward --> WO_Mix
    WO_Mix --> JC_Mix
    SE_Hold --> CI

    %% Edges - QC
    SE_Inward --> CI
    CI --> CI_Check
    CI_Check -- Pass --> SE_Approve
    CI_Check -- Fail --> SE_Reject
    SE_Approve --> BDE

    %% Edges - Blanking
    BDE --> BDE_Val
    BDE_Val --> SE_Blank
    BDE_Val --> AM_Blank
    AM_Blank --> BBIE
    
    CBT --> SE_CBT

    BBIE --> BBIE_Choice
    BBIE_Choice -- No --> AM_Inward
    BBIE_Choice -- Yes --> SE_Inward_Repack
    AM_Inward --> BBI

    %% Edges - Planning
    WP --> WP_Gen
    WP_Gen --> WO_Mould
    WP_Gen --> JC_Mould
    WO_Mould -.-> MPE
    JC_Mould -.-> MPE

    %% Edges - Moulding
    BBI --> BBI_Val
    BBI_Val --> MPE
    MPE --> MPE_Val
    MPE_Val -- OK --> MPE_Entry
    MPE_Entry --> SE_Mould
    MPE_Entry --> JC_Update

    %% Edges - Finishing
    SE_Mould --> DDE
    DDE --> SE_Def_Out
    SE_Def_Out --> RDE
    RDE --> SE_Def_In
    SE_Def_In --> LRT
    LRT --> LRT_Pack

    %% Styles
    style DCR fill:#e1f5fe,stroke:#01579b
    style CI fill:#e8f5e9,stroke:#2e7d32
    style BDE fill:#fff3e0,stroke:#ef6c00
    style WP fill:#f3e5f5,stroke:#7b1fa2
    style MPE fill:#ffebee,stroke:#c62828
    style LRT fill:#e0f7fa,stroke:#006064
```

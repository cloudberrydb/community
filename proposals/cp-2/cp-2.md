> Title: [Proposal] Refactor ORCA to support user-defined access method in pg<br>
> Date: 08/02, 2023
> GitHub Discussions: https://github.com/orgs/cloudberrydb/discussions/113

### Proposers
@wfnuser [Qinghao Huang]

### Proposal Status
Under discussion

### Abstract
Introduce a novel infrastructure to enable ORCA's support for any user-defined access method.  

### Motivation
Just as Postgres supports user-defined access methods through the creation of extensions, it is imperative for ORCA to follow suit.  

### Background
In Postgres, we possess a well-structured abstraction of an access method that enables us to accommodate any user-defined access approach. For introducing novel index types like the 'bloom index', developers simply need to implement the API structure 'IndexAmRoutine' and establish the new index access method as an extension. Subsequently, Postgres can automatically produce associated index scan paths within the optimizer and invoke the method for the fresh index type implemented by the extension provider if the path is selected in the executor.  
However, this narrative takes a different turn when it comes to ORCA.  
Firstly, all the index types supported in ORCA are predefined within its implementation. Any time we are translating `relcache` to `dxl` or evaluating the applicability of an index, these processes are hardcoded into the related logic.  

``` cpp
		/* Definition */
		enum EmdindexType
		{
		    EmdindBtree,   // btree
		    EmdindBitmap,  // bitmap
		    EmdindGist,    // gist using btree or bitmap
		    EmdindGin,     // gin using btree or bitmap
		    EmdindBrin,    // brin
		    EmdindHash,    // hash
		    EmdindSentinel
		};
		
		/* Example 1 */
		switch (index_rel->rd_rel->relam)
		{
			case BTREE_AM_OID:
				index_type = IMDIndex::EmdindBtree;
				break;
			case BITMAP_AM_OID:
				index_type = IMDIndex::EmdindBitmap;
				break;
			case BRIN_AM_OID:
				index_type = IMDIndex::EmdindBrin;
				break;
			case GIN_AM_OID:
				index_type = IMDIndex::EmdindGin;
				break;
			case GIST_AM_OID:
				index_type = IMDIndex::EmdindGist;
				break;
			case HASH_AM_OID:
				index_type = IMDIndex::EmdindHash;
				break;
			default:
				GPOS_RAISE(gpdxl::ExmaMD, gpdxl::ExmiMDObjUnsupported,
						   GPOS_WSZ_LIT("Index access method"));
		}
		
		/* Example 2 */
		// index expressions and index constraints not supported
		return gpdb::HeapAttIsNull(tup, Anum_pg_index_indexprs) &&
			   gpdb::HeapAttIsNull(tup, Anum_pg_index_indpred) &&
			   index_rel->rd_index->indisvalid &&
			   (BTREE_AM_OID == index_rel->rd_rel->relam ||
				BITMAP_AM_OID == index_rel->rd_rel->relam ||
				GIST_AM_OID == index_rel->rd_rel->relam ||
				GIN_AM_OID == index_rel->rd_rel->relam ||
				BRIN_AM_OID == index_rel->rd_rel->relam ||
				HASH_AM_OID == index_rel->rd_rel->relam);
		
		/* Example 3 */
		BOOL gin_or_gist_or_brin = (pmdindex->IndexType() == IMDIndex::EmdindGist ||
		                            pmdindex->IndexType() == IMDIndex::EmdindGin ||
		                            pmdindex->IndexType() == IMDIndex::EmdindBrin);		
if (cmptype == IMDType::EcmptNEq || cmptype == IMDType::EcmptIDF ||
		    (cmptype == IMDType::EcmptOther &&
		    !gin_or_gist_or_brin) ||  // only GIN/GiST/BRIN indexes with a comparison type other are ok
		    (gin_or_gist_or_brin &&
		    pexprScalar->Arity() <
		        2))  // we do not support unary index expressions for GIN/GiST/BRIN indexes
		{
		    return nullptr;
		}			  
```

Secondly, ORCA operates on a distinct cost model, which implies that the costs computed in the Postgres Optimizer cannot be directly compared with those calculated in ORCA. Therefore, devising a method to compute costs in ORCA for user-defined indexes stands as another challenge that needs to be resolved to facilitate user-defined indexes within ORCA. 

### Implementation (WIP)
The solution involves multiple components:  

#### PG_AM
To extend support for ORCA and other third-party optimizers, we introduce a new cost estimate function within `IndexAmRoutine`. This function should be optional. If extension providers wish to enable it for ORCA, they must be responsible for implementing the cost estimate logic. If not provided, there should be no adverse effects on the Postgres Optimizer.  

``` c
		  typedef struct IndexAmRoutine
		  {
		  	NodeTag		type;
		  	...
		  
		  	/* third-party am cost estimate functions */
		  	AmOrcaCostEstimateFunc amorcacostestimate; /* can be NULL */
		  } IndexAmRoutine;
		  
		  typedef double (*AmOrcaCostEstimateFunc) (CostInfo* ci);
```

Potentially, modifications may be needed in `IndexAmRoutine` to provide additional information required by ORCA, though specifics remain undiscovered.
For CostInfo please refer to ###Cost Model (SPIKING)

#### DXL translation parts (WIP)
Introduce a new index type called `UserDefinedIndex` for non-predefined indexes.  

``` cpp
			      /* Definition */
			      enum EmdindexType
			      {
			          EmdindBtree,   // btree
			          EmdindBitmap,  // bitmap
			          EmdindGist,    // gist using btree or bitmap
			          EmdindGin,     // gin using btree or bitmap
			          EmdindBrin,    // brin
			          EmdindHash,    // hash
			          EmdindUserDefined, // user defined
			          EmdindSentinel
			      };
```

Remove hardcoded index type checks such as `IsIndexSupported`. All indexes are supported if they provide the implementation of `amorcacostestimate`. 

#### Metadata (SPIKING)
FIndexApplicable:  
- During the evaluation of `FIndexApplicable`, decisions must be made regarding whether the new index can support Btree or Bitmap indexes.  
- Determining which types of comparison are supported for different indexes.
- Adding fields to indicate the aforementioned considerations.  
- Keeping a safe plan if certain decisions haven't been made yet.  

On the ORCA side, a suitable location needs to be identified for attaching the user-defined cost estimate function. `CMDIndexGPDB` appears to be a natural choice, since the access method is linked to index relations in Postgres:  

``` cpp
			  	CMDIndexGPDB *index = GPOS_NEW(mp) CMDIndexGPDB(
			  		mp, mdid_index, mdname, index_clustered, index_partitioned, index_type,
			  		mdid_item_type, index_key_cols_array, included_cols, op_families_mdids,
			  		nullptr,  // mdpart_constraint
			  		child_index_oids,
			  		index_rel->rd_indam->amorcacostestimate);
```

This enables us to access the function by using the `RetrieveIndex` method at any point where we have the relevant `MDId`.  

#### Cost Model (SPIKING)
Currently, there's a reliance on switch and case style implementation to accommodate distinct cost models for various operators in ORCA. And associating cost models with operators is the direction we're heading. A potential long-term objective might look like the following:  

``` cpp
			  CCost
			  CCostModelGPDB::Cost(
			  	CExpressionHandle &exprhdl,	 // handle gives access to expression properties
			  	const SCostingInfo *pci) const
			  {
			  	GPOS_ASSERT(nullptr != pci);
			  	COperator::EOperatorId op_id = exprhdl.Pop()->Eopid();
			  	// All infomation the customized cost estimate function need to know
			  	CostInfo ci = new CostInfo();
			  	return exprhdl.Pop().Cost(ci);
			    
			  	/* CURRENT IMPLEMENTATION */
			  	// switch (op_id)
			  	// {
			  	// 	default:
			  	// 	{
			  	// 		// FIXME: macro this?
			  	// 		__builtin_unreachable();
			  	// 	}
			  	// 	case COperator::EopPhysicalTableScan:
			  	// 	case COperator::EopPhysicalDynamicTableScan:
			  	// 	case COperator::EopPhysicalExternalScan:
			  	// 	{
			  	// 		return CostScan(m_mp, exprhdl, this, pci);
			  	// 	}
			  
			  	// 	case COperator::EopPhysicalFilter:
			  	// 	{
			  	// 		return CostFilter(m_mp, exprhdl, this, pci);
			  	// 	}
			  
			  	// 	case COperator::EopPhysicalIndexOnlyScan:
			  	// 	{
			  	// 		return CostIndexOnlyScan(m_mp, exprhdl, this, pci);
			  	// 	}
			  	// } 
			  }
			  
			  
```

However, this is a substantial change that cannot be implemented in a single commit. The initial step would be to support two operators (`PhysicalIndexScan` and `PhysicalTableScan`) in a polymorphism style, not switch case style.  
Initial implementation could look like:  

``` cpp
			  CCost
			  CCostModelGPDB::Cost(
			  	CExpressionHandle &exprhdl,	 // handle gives access to expression properties
			  	const SCostingInfo *pci) const
			  {
			  	GPOS_ASSERT(nullptr != pci);
			  
			  	COperator::EOperatorId op_id = exprhdl.Pop()->Eopid();
			  
			  	if (op_id == COperator::EopPhysicalIndexScan)
			  	{
			  		CPhysicalIndexScan *pop = (CPhysicalIndexScan*) exprhdl.Pop();
			  		CMDAccessor *md_accessor = COptCtxt::PoctxtFromTLS()->Pmda();
			  		const IMDIndex *pmdindex = md_accessor->RetrieveIndex(pop->Pindexdesc()->MDId());
			  		if (pmdindex->OrcaCostEsitmate() != NULL)
			  		{
			  			return CostUserDefinedIndex(m_mp, exprhdl, this, pci, pmdindex->OrcaCostEsitmate());
			  		}
			  	}
			  }
			  
			  // main job for this function is to create CostInfo for this particular operator
			  // the info for different operator seems to be different
			  CCost
			  CCostModelGPDB::CostUserDefinedIndex(CMemoryPool *,  // mp
			  							  CExpressionHandle &exprhdl,
			  							  const CCostModelGPDB *pcmgpdb,
			  							  const SCostingInfo *pci,
			  							  AmOrcaCostEstimateFunc cef)
			  {
			  	COperator *pop = exprhdl.Pop();
			  	CostInfo ci;
			  
			  	/* ORCA_AM_TODO: the way to get table width is currently related to pop. */
			  	ci.dTableWidth =
			  		CPhysicalScan::PopConvert(pop)->PstatsBaseTable()->Width().Get();
			  
			  	ci.dIndexFilterCostUnit =
			  		pcmgpdb->GetCostModelParams()
			  			->PcpLookup(CCostModelParamsGPDB::EcpIndexFilterCostUnit)
			  			->Get().Get();
			  	ci.dIndexScanTupCostUnit =
			  		pcmgpdb->GetCostModelParams()
			  			->PcpLookup(CCostModelParamsGPDB::EcpIndexScanTupCostUnit)
			  			->Get().Get();
			  	ci.dIndexScanTupRandomFactor =
			  		pcmgpdb->GetCostModelParams()
			  			->PcpLookup(CCostModelParamsGPDB::EcpIndexScanTupRandomFactor)
			  			->Get().Get();
			  
			  	GPOS_ASSERT(0 < ci.dIndexFilterCostUnit);
			  	GPOS_ASSERT(0 < ci.dIndexScanTupCostUnit);
			  	GPOS_ASSERT(0 < ci.dIndexScanTupRandomFactor);
			  
			  	ci.dRowsIndex = pci->Rows();
			  	ci.dNumRebinds = pci->NumRebinds();
			  
			  	ci.ulIndexKeys = CPhysicalIndexScan::PopConvert(pop)->Pindexdesc()->Keys();
			  
			  	return CCost(cef(&ci));
			  }
```

It's also worth noting that the `CostInfo` structure should be meticulously designed to ensure that it's universally applicable across different operators. We intend to maintain this structure in the PostgreSQL codebase using the C language. By doing so, ORCA and other third-party optimizers can rely on it as a dependable reference. An illustrative sample structure is as follows:  

``` c
			  typedef struct CostInfo
			  {
			      double dTableWidth;
			      double dIndexFilterCostUnit;
			      double dIndexScanTupCostUnit;
			      double dIndexScanTupRandomFactor;
			      double dRowsIndex;
			      double dNumRebinds;
			      uint32_t ulIndexKeys;
			  } CostInfo;
```

#### Cost Weight (Unspecified)
Determining the cost weight for a specific operator remains uncertain. Decisions should be based on existing methodologies, but we don't have enough info about it.  

``` c
		  double
		  hashorcacostestimate(CostInfo* ci)
		  {
		  	// We don't need random IO cost. But we have some other costs. How to decide it.
		  	double dCostPerIndexRow = ci->dTableWidth * ci->dIndexScanTupCostUnit;
		  	return ci->dNumRebinds *
		  				 (ci->dRowsIndex * dCostPerIndexRow);
		  }
```

This [paper - Counting, Enumerating, and Sampling of Execution Plans
in a Cost-Based Query Optimizer](https://sigmodrecord.org/publications/sigmodRecord/0006/pdfs/Counting,%20Enumerating,%20and%20Sampling%20of%20Execution%20Plans%20in%20a%20Cost-Based%20Query%20Optimizer.pdf) might give us some clues.

#### Property Enforcement (Unspecified)
Currently, only `btscan` has the order property among indexes. For other index scans, we can assume that they don't differ in properties from table scans.  

#### Bitmap Index Scan & Index Only Scan (TODO)
To facilitate the integration of bitmap index scans and index-only scans, it's imperative to introduce specific flags that allow customized index providers to determine whether they are capable of supporting these types of scans. 

Presently, the plan generated by ORCA appears to include information only for higher-level operators such as the bitmap heap scan, while the cost information pertaining to lower-level bitmap index scans seems to be absent. This discrepancy needs to be addressed in order to provide a comprehensive view of the plan's cost estimation, especially concerning the individual index scans and their associated costs.

```sql
postgres=# explain select * from t1 where a = 97 or a = 98;
                                    QUERY PLAN
-----------------------------------------------------------------------------------
 Gather Motion 3:1  (slice1; segments: 3)  (cost=0.00..431.51 rows=1898 width=4)
   ->  Bitmap Heap Scan on t1  (cost=0.00..431.48 rows=633 width=4)
         Recheck Cond: ((a = 97) OR (a = 98))
         ->  BitmapOr  (cost=0.00..0.00 rows=0 width=0)
               ->  Bitmap Index Scan on t1_a_idx  (cost=0.00..0.00 rows=0 width=0)
                     Index Cond: (a = 97)
               ->  Bitmap Index Scan on t1_a_idx  (cost=0.00..0.00 rows=0 width=0)
                     Index Cond: (a = 98)
 Optimizer: Pivotal Optimizer (GPORCA)
(9 rows)

postgres=# drop index t1_a_idx;
DROP INDEX
postgres=# explain select * from t1 where a = 97 or a = 98;
                                       QUERY PLAN
----------------------------------------------------------------------------------------
 Gather Motion 3:1  (slice1; segments: 3)  (cost=0.00..431.51 rows=1898 width=4)
   ->  Bitmap Heap Scan on t1  (cost=0.00..431.48 rows=633 width=4)
         Recheck Cond: ((a = 97) OR (a = 98))
         ->  BitmapOr  (cost=0.00..0.00 rows=0 width=0)
               ->  Bitmap Index Scan on hash_index_t1  (cost=0.00..0.00 rows=0 width=0)
                     Index Cond: (a = 97)
               ->  Bitmap Index Scan on hash_index_t1  (cost=0.00..0.00 rows=0 width=0)
                     Index Cond: (a = 98)
 Optimizer: Pivotal Optimizer (GPORCA)
(9 rows)
```

#### TESTs (WIP)
Relevant tests must be developed and added to validate the changes.  

### Alternatives
- Utilizing user-defined functions within SQL as opposed to linking functions to IndexAmRoutine.  
- Initially retrieving the Index, then associating `ORCACostEstimateFunc` with `CExpression` and `CGroupExpression`, and triggering it from expressions during cost computation, rather than re-retrieving the index.  
- Introducing new operators for `UserDefinedIndexScan` and `UserDefinedTableScan`, deviating from the use of the same operators as previously.  
- Developing an extension by informing developers about ORCA information and making it a dependency. This could allow direct injection of serialized cost models written in C++ to Postgres. If this approach proves unfeasible, referencing certain header files from ORCA might still be possible (so that we can use cost weights from orca directly).  

### Related issues
[#80](https://github.com/cloudberrydb/cloudberrydb/issues/80)

### Are you willing to submit a PR?

- [X] Yes I am willing to submit a PR!
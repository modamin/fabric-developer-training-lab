# Cleanup (5 min)

1. **Delete the workspace** (removes everything at once):
   `Meridian-HR-Lab` → **Workspace settings → Remove this workspace**. Deletes the
   lakehouse, notebooks, pipeline, semantic model, and data agent together.

2. To keep the workspace but free storage, drop the data:
   ```python
   for sch in ["bronze","silver","gold"]:
       for t in spark.sql(f"SHOW TABLES IN {sch}").collect():
           spark.sql(f"DROP TABLE {sch}.{t.tableName}")
   ```
   and delete `Files/landing/` in the lakehouse.

3. **OAuth fixed-identity connection:** remove
   `conn_meridian_onelake_oauth` under **Manage connections and gateways**. The
   lab uses your student account, so there is no service principal or managed
   identity to delete.

4. **Capacity:** if you started a trial/paid F-capacity just for this lab, pause
   or delete it in the Azure portal so it stops billing.

The OAuth cloud connection in step 3 is the only lab artifact outside the
workspace.

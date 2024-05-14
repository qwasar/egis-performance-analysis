# EGIS Performance tips
## Direct use of selected objects in GeoMedia
There are places where the selection object from GM is used to get only the ID of the object, which is then used to summon the full object again. 
```CSharp
        var selectedFeatures = GeoMediaMapWindowManager.GetSelectedObjectFromMapWindow();
        if (selectedFeatures.First().FeatureClassInfo.FCBaseDefinition.Name == GLEISKANTE.TableName) // Stationierung
        {
            var gkaIds = new List<long>();
            foreach (var feature in selectedFeatures)
            {
                gkaIds.Add(Convert.ToInt64(feature.PrimaryFieldValue));

// ... later
        var gkaFeature = EGISHelper.GetFeatureByName(GLEISKANTE.TableName, gkaIds);
        var gknIds = EGISHelper.GetAttributeValues<long>(gkaFeature, GLEISKANTE.GKN_ID_VON)
            .Concat(EGISHelper.GetAttributeValues<long>(gkaFeature, GLEISKANTE.GKN_ID_BIS));
```
But the feature's attributes are accessible directly on the selection object: 
```CSharp
        var gknIdVon = selectedFeatures.First().FeatureAttributes.FindByName("GLEISKANTE.GKN_ID_VON").Value;
        var gknIdBis = selectedFeatures.First().FeatureAttributes.FindByName("GLEISKANTE.GKN_ID_BIS").Value;
```

## FunctionalAttributesPipe and functional attributes
Instead of setting up a FunctionalAttributesPipe to get the first and last point of a polyline like this:
```CSharp
        public const string EXPRESSION_FUNC_ATTR_START = "STARTPOINT(FILTERLINEAR(Input.";
        public const string EXPRESSION_FUNC_ATTR_END = "ENDPOINT(FILTERLINEAR(Input.";
...
        var fullFuncAttrExpr = funcAttrExpr + GLEISKANTE.GEO_COMPOUND + "))";
        var gkaGkns = GMHelper.FunktionsattributErzeugen(gkaFeature.Recordset, fullFuncAttrExpr, funcAttrOutColumn);
        geometry = gkaGkns.GFields[funcAttrOutColumn].Value;
```
direct property access of the geometry object can be probably used:
```CSharp
        if (gkaGeometry is PBasic.PolylineGeometry polyline)
        {
            // OBS!! 1-based index
            startPktGeometry = new PBasic.PointGeometryClass() { Origin = polyline.Points.Item(1) };
            endPktGeometry = new PBasic.PointGeometry { Origin = polyline.Points.Item(polyline.Points.Count) };
        }
```

## Spatial search (buffer + intersection pipes)
For finding the GKN, WA etc. features in a close proximity of an Anf/End point of a GKA rather complex code is used. First a BufferPipe around the point is created and then a SpatialIntersectionPipe is used to find features in the target feature class that are contained in the specified buffer. This approach is quite resource heavy and usually takes several seconds.
```CSharp
        // Bufferzone around StartPoint GKA 
        gkaBuffer = GMHelper.ErstelleBufferPipe((GDO.GRecordset)gkaGkns, funcAttrOutColumn, radius);

        if (Feature.IsEmpty(gknFeature) == false)
        {
          var verschneideRS = GMHelper.VerschneideRecordsets(
                gkaBuffer,
                GMHelper.BUFFER_GEOMETRY_COLUMN,
                gknFeature.Recordset,
                DBGDI.GLEISKNOTEN.GEO_PUNKT,
                PPipe.SQConstants.gmsqContains,
                GMHelper.INTERSECTION_GEOMETRY_COLUMN);

          // Intersection to find all possible GKN features to reference StartPoint GKA
          using (var intersectionRs = SmartGRecordset.Create(verschneideRS))
```
Alternative approach could be to utilize a SpatialFilter on an OriginatingPipe. According to [Pavel Krejcir](https://supportsi.hexagon.com/help/s/question/0D52o0000CMbUqVCQV/how-to-use-the-spatialquerypipe-in-geomedia-automation-to-perform-a-spatial-query?language=es) this should be the fastest way how to do a spatial search with pipes. 
```CSharp
          var op = new OriginatingPipe();
          op.Table = "DBGDI.GLEISKNOTEN";
          op.SpatialFilter = new PBasic.PolylineGeometry();
          op.SpatialOperator = SpatialOperator.Contains;
```
But it can be made even faster by utilizating the __SDO_WITHIN_DISTANCE__ spatial operator in Oracle
```CSharp
          public static Feature GetFeaturesWithinDistance(IProxyDataServerFeatureClassInfo fci, double x, double y, ICustomDependencyAutomationHelper helper)
          {
              var geometryColumn = fci.GeometryKey.Name;
      
              var sdoPoint = FormattableString.Invariant($"SDO_GEOMETRY(3001, {srid}, SDO_POINT_TYPE({x}, {y}, 0), NULL, NULL)");
              var whereClause = FormattableString.Invariant($"SDO_WITHIN_DISTANCE({geometryColumn}, {sdoPoint}, 'distance={radius} unit=meter') = 'TRUE'");
      
              var features = helper.GetCursorFeature(fci, whereClause);
              return features;
          }
```
Question is if and if so, then under what circumstances, such technique could have been used. In ID101 the ```DATE_TO = date '9000-01-01' and ACTIVEJOBID = -1``` clause had to be added to all queries to make them work only on applicable features. How such conditions are applied in EGIS? I have found ```TechnicalValidityHelper.BuildTechnicalValidityFilter``` but it doesn't seem to work at all...

## Int32 vs. Int64 as database entities ID property
It looks like in the codebase it is assumed that ID fields are of the ```long``` type (i.e. Int64). However in GDO, the corresponding field type (gdbLong = 4) represents actually Int32. Thus converting or casting those values to long is redundant. It means that from the Oracle's NUMBER(38) data type only ~9 digits is usable in GeoMedia too.


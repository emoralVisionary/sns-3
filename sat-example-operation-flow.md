# sat-http-example Function-call (Operation) Flow-chart

```C++
NS_LOG_COMPONENT_DEFINE ("sat-http-example");

int main (int argc, char *argv[])
{
  std::string scenario = "simple";
  double duration = 20; // double duration = 1000;
  SatHelper::PreDefinedScenario_t satScenario = SatHelper::SIMPLE; // Declare a variable satScenario that has enum type defined in class SatHelper

  Config::SetDefault ("ns3::SatHelper::ScenarioCreationTraceEnabled", BooleanValue (true));

  auto simulationHelper = CreateObject<SimulationHelper> ("example-http");
  Config::SetDefault ("ns3::SatEnvVariables::EnableSimulationOutputOverwrite", BooleanValue (true));

  // read command line parameters given by user
  CommandLine cmd;
  cmd.AddValue ("scenario", "Test scenario to use. (simple, larger or full)",
                scenario);
  cmd.AddValue ("duration", "Simulation duration (in seconds)",
                duration);
  simulationHelper->AddDefaultUiArguments (cmd);
  cmd.Parse (argc, argv);

  if (scenario == "larger")
    {
      satScenario = SatHelper::LARGER;
    }
  else if (scenario == "full")
    {
      satScenario = SatHelper::FULL;
    }

  /// Set simulation output details
  simulationHelper->SetSimulationTime (duration);
  simulationHelper->SetOutputTag (scenario);


  LogComponentEnableAll (LOG_PREFIX_ALL);
  LogComponentEnable ("HttpClient", LOG_LEVEL_ALL);
  LogComponentEnable ("HttpServer", LOG_LEVEL_ALL);
  //LogComponentEnable ("sat-http-example", LOG_LEVEL_ALL);

  // remove next line from comments to run real time simulation
  // GlobalValue::Bind ("SimulatorImplementationType", StringValue ("ns3::RealtimeSimulatorImpl"));

  // Creating the reference system. Note, currently the satellite module supports
  // only one reference system, which is named as "Scenario72". The string is utilized
  // in mapping the scenario to the needed reference system configuration files. Arbitrary
  // scenario name results in fatal error.
  Ptr<SatHelper> helper = simulationHelper->CreateSatScenario (satScenario);

  // get users
  NodeContainer utUsers = helper->GetUtUsers ();
  NodeContainer gwUsers = helper->GetGwUsers ();

  ThreeGppHttpHelper httpHelper;
  httpHelper.InstallUsingIpv4 (gwUsers.Get (0), utUsers);
  httpHelper.GetServer ().Start (Seconds (1.0));

  auto apps = httpHelper.GetClients ();
  apps.Start (Seconds (3.0));

  uint32_t i = 0;
  std::vector<Ptr<ClientRxTracePlot> > plots;
  for (auto app = apps.Begin (); app != apps.End (); app++, i++)
  {
    std::stringstream plotName;
    plotName << "3GPP-HTTP-client-" << i << "-trace";
  	plots.push_back (CreateObject<ClientRxTracePlot> (*app, plotName.str ()));
  }

  NS_LOG_INFO ("--- sat-http-example ---");
  NS_LOG_INFO ("  Scenario used: " << scenario);
  NS_LOG_INFO ("  ");

  simulationHelper->RunSimulation();
        
  plots.clear();

  return 0;

} // end of `int main (int argc, char *argv[])`

```
auto simulationHelper = CreateObject<SimulationHelper> ("example-http"); at line 14

```C++
template <typename T>
Ptr<T> CreateObject (void)
{
  return CompleteConstruct (new T ());
}
```
> `T ::= SimulationHelper`
<br> line 93 => `return CompleteConstruct (new SimulationHelper ("example-http"));` <br>
: with argument "example-http", Call the contructor of SimulationHelper 
<br>=> Created SimulationHelper instance is delivered to func. CompleteConstruct as an argument => return the pointer that refer the instance of class T

```C++ 
template <typename T>
Ptr<T> CompleteConstruct (T *object)
{
  object->SetTypeId (T::GetTypeId ());
  object->Object::Construct (AttributeConstructionList ());
  return Ptr<T> (object, false);
}
```
At lin 105, `Object::Construct (...)` is called 

```C++
void
Object::Construct (const AttributeConstructionList &attributes)
{
  NS_LOG_FUNCTION (this << &attributes);
  ConstructSelf (attributes);
}
```
> ContructSelf is a function of a class ObjectBase. It seems to be a function related to the construction of object in ns-3 (in ns3 core).
<br> _cf. sns3/bake/source/ns-3.29/src/core/model/object-base.cc_
<br> \* In general ns-3 moreover sns-3, is ConstructSelf(...) also called when constructing an object? => Check if possible 

```C++
Ptr<SatHelper> SimulationHelper::CreateSatScenario (SatHelper::PreDefinedScenario_t scenario)
{
  NS_LOG_FUNCTION (this);

  std::stringstream ss;
  ss << "Created scenario: " << std::endl;

  // Set final output path
  SetupOutputPath ();

  m_satHelper = CreateObject<SatHelper> ();

  // Set UT position allocators, if any
  if (!m_enableInputFileUtListPositions)
    {
      if (m_commonUtPositions) m_satHelper->SetCustomUtPositionAllocator (m_commonUtPositions);
      for (auto it : m_utPositionsByBeam) m_satHelper->SetUtPositionAllocatorForBeam (it.first, it.second);
    }

  m_satHelper->CreatePredefinedScenario (scenario);

  NS_LOG_INFO (ss.str ());
  return m_satHelper;
}  
```
> At line 134, ` m_satHelper = CreateObject<SatHelper> ();` : Create a SatHelper instance
<br> So, We need to comprehend a operation of `SatHelper::SatHelper()`

<br>

```C++
SatHelper::SatHelper ()
  : m_rtnConfFileName ("Scenario72RtnConf.txt"),
    m_fwdConfFileName ("Scenario72FwdConf.txt"),
    m_gwPosFileName ("Scenario72GwPos.txt"),
    m_geoPosFileName ("Scenario72GeoPos.txt"),
    m_waveformConfFileName ("dvbRcs2Waveforms.txt"),
    m_scenarioCreated (false),
    m_creationTraces (false),
    m_detailedCreationTraces (false),
    m_packetTraces (false),
    m_utsInBeam (0),
    m_gwUsers (0),
    m_utUsers (0),
	m_utPositionsByBeam ()
{
  NS_LOG_FUNCTION (this);

  // Check the meaning and the function of below two lines
  ObjectBase::ConstructSelf(AttributeConstructionList ());
  Singleton<SatEnvVariables>::Get ()->Initialize ();

  // SatConf instance creation!
  m_satConf = CreateObject<SatConf> (); // Nothing done, just set the default values
  // Critical Function!!
  m_satConf->Initialize (m_rtnConfFileName,
                         m_fwdConfFileName,
                         m_gwPosFileName,
                         m_geoPosFileName,
                         m_waveformConfFileName);

  // Create antenna gain patterns
  m_antennaGainPatterns = CreateObject<SatAntennaGainPatternContainer> ();

  // create Geo Satellite node, set mobility to it
  Ptr<Node> geoSatNode = CreateObject<Node> ();
  SetGeoSatMobility (geoSatNode);

  m_beamHelper = CreateObject<SatBeamHelper> (geoSatNode,
                                              MakeCallback (&SatConf::GetCarrierBandwidthHz, m_satConf),
                                              m_satConf->GetRtnLinkCarrierCount (),
                                              m_satConf->GetFwdLinkCarrierCount (),
                                              m_satConf->GetSuperframeSeq ());

  Ptr<SatRtnLinkTime> rtnTime = Singleton<SatRtnLinkTime>::Get ();
  rtnTime->Initialize (m_satConf->GetSuperframeSeq ());

  SatBeamHelper::CarrierFreqConverter converterCb = MakeCallback (&SatConf::GetCarrierFrequencyHz, m_satConf);
  m_beamHelper->SetAttribute ("CarrierFrequencyConverter", CallbackValue (converterCb) );

  m_userHelper = CreateObject<SatUserHelper> ();

  // Set the antenna patterns to beam helper
  m_beamHelper->SetAntennaGainPatterns (m_antennaGainPatterns);
}
```
> \* We should comprehend SatConf::Initialize(...)

```C++
void SatConf::Initialize (std::string rtnConf,
                          std::string fwdConf,
                          std::string gwPos,
                          std::string satPos,
                          std::string wfConf)
{
  NS_LOG_FUNCTION (this);

  std::string dataPath = Singleton<SatEnvVariables>::Get ()->LocateDataDirectory () + "/";

  // Load satellite configuration file
  m_rtnConf = LoadSatConf (dataPath + rtnConf);
  m_fwdConf = LoadSatConf (dataPath + fwdConf);

  NS_ASSERT (m_rtnConf.size () == m_fwdConf.size ());
  m_beamCount = m_rtnConf.size ();

  // Load GW positions
  LoadPositions (dataPath + gwPos, m_gwPositions);

  // Load UT positions
  LoadPositions (dataPath + m_utPositionInputFileName, m_utPositions);

  // Load satellite position
  LoadPositions (dataPath + satPos, m_geoSatPosition);

  Configure (dataPath + wfConf);
}
```
> `SatConf::LoadPositions(...)` loads the pre-defined positions (in a corresponding file) of each element => Check the error condition when setting positions

```C++
void
SatConf::LoadPositions (std::string filePathName, PositionContainer_t& container)
{
  NS_LOG_FUNCTION (this << filePathName);

  // READ FROM THE SPECIFIED INPUT FILE
  std::ifstream *ifs = OpenFile (filePathName);

  double lat, lon, alt;
  *ifs >> lat >> lon >> alt;

  while (ifs->good ())
    {
      NS_LOG_DEBUG (this <<
                    " latitude [deg] = " << lat <<
                    ", longitude [deg] = " << lon <<
                    ", altitude [m] = ");

      // Store the values
      GeoCoordinate coord (lat, lon, alt);
      container.push_back (coord);

      // get next row
      *ifs >> lat >> lon >> alt;
    }

  ifs->close ();
  delete ifs;
}
```
> At line 265, Call `GeoCoordinate coord (lat, lon, alt)` then, 
<br> `GeoCoordinate::GeoCoordinate(...)` calls `GeoCoordinate::Construct (latitude, longitude, altitude);` 
<br> Then... let's analyze...

```C++
void GeoCoordinate::Construct (double latitude, double longitude, double altitude)
{
  NS_LOG_FUNCTION (this << latitude << longitude << altitude);

  // Before Initialize(), there is a test for checking a value validity
  if ( IsValidLatitude (latitude) == false)
    {
      NS_FATAL_ERROR ("Invalid latitude!!!");
    }

  if (IsValidLongtitude (longitude) == false)
    {
      NS_FATAL_ERROR ("Invalid longitude!!!");
    }

  if (IsValidAltitude (altitude, m_refEllipsoid) == false)
    {
      NS_FATAL_ERROR ("Invalid altitude!!!");
    }

  Initialize ();

  m_latitude = latitude;
  m_longitude = longitude;
  m_altitude = altitude;
}
```
> Analyze `IsValid*(...)` => Check whether we could modify a value of GW's altitude to what we want (LEO Sat's altitude)

```C++
bool GeoCoordinate::IsValidAltitude (double altitude, ReferenceEllipsoid_t refEllipsoid)
{
  double polarRadius = NAN;

  switch ( refEllipsoid )
    {
    case SPHERE:
      polarRadius = GeoCoordinate::polarRadius_sphere;
      break;

    case WGS84:
      polarRadius = GeoCoordinate::polarRadius_wgs84;
      break;

    case GRS80:
      polarRadius = GeoCoordinate::polarRadius_grs80;
      break;

    default:
      NS_FATAL_ERROR ("Invalid polar radius value!!!");
      break;
    }

  return ( (polarRadius + altitude) >= 0 );
}
```

```C++
void GeoCoordinate::Initialize ()
{
  NS_LOG_FUNCTION (this);

  switch ( m_refEllipsoid )
    {
    case SPHERE:
      m_polarRadius = GeoCoordinate::polarRadius_sphere;
      break;

    case WGS84:
      m_polarRadius = GeoCoordinate::polarRadius_wgs84;
      break;

    case GRS80:
      m_polarRadius = GeoCoordinate::polarRadius_grs80;
      break;

    default:
      NS_FATAL_ERROR ("Invalid Reference Ellipsoid!!!");
      break;
    }

  m_equatorRadius = GeoCoordinate::equatorRadius;
  m_e2Param = ( ( m_equatorRadius * m_equatorRadius ) - ( m_polarRadius * m_polarRadius ) ) / (m_equatorRadius * m_equatorRadius );
}
```

```C++
void
GeoCoordinate::Construct (double latitude, double longitude, double altitude)
{
  NS_LOG_FUNCTION (this << latitude << longitude << altitude);

  if ( IsValidLatitude (latitude) == false)
    {
      NS_FATAL_ERROR ("Invalid latitude!!!");
    }

  if (IsValidLongtitude (longitude) == false)
    {
      NS_FATAL_ERROR ("Invalid longitude!!!");
    }

  if (IsValidAltitude (altitude, m_refEllipsoid) == false)
    {
      NS_FATAL_ERROR ("Invalid altitude!!!");
    }

  Initialize ();

  m_latitude = latitude;
  m_longitude = longitude;
  m_altitude = altitude;
}
```



--- 
```C++
void SatHelper::CreatePredefinedScenario (PreDefinedScenario_t scenario)
{
  NS_LOG_FUNCTION (this);

  switch (scenario)
    {
    case SIMPLE:
      CreateSimpleScenario ();
      break;

    case LARGER:
      CreateLargerScenario ();
      break;

    case FULL:
      CreateFullScenario ();
      break;

    default:
      NS_FATAL_ERROR ("Not supported predefined scenario.");
      break;
    }
}
```
* if `scenario = SIMPLE` then goto CreateSimpleScenario () *

```C++
void SatHelper::CreateSimpleScenario ()
{
  NS_LOG_FUNCTION (this);

  SatBeamUserInfo beamInfo = SatBeamUserInfo (1,1);
  BeamUserInfoMap_t beamUserInfos;
  beamUserInfos[8] = beamInfo;

  DoCreateScenario (beamUserInfos, 1);

  m_creationSummaryTrace ("*** Simple Scenario Creation Summary ***");
}
```
Every scenario calls SatHelper:DoCreateScenario(...)

```C++
void SatHelper::DoCreateScenario (BeamUserInfoMap_t& beamInfos, uint32_t gwUsers)
{
  NS_LOG_FUNCTION (this);

  if (m_scenarioCreated)
    {
      Singleton<SatLog>::Get ()->AddToLog (SatLog::LOG_WARNING, "", "Scenario tried to re-create with SatHelper. Creation can be done only once.");
    }
  else
    {
      SetNetworkAddresses (beamInfos, gwUsers);

      if (m_creationTraces)
        {
          EnableCreationTraces ();
        }

      InternetStackHelper internet;

      // create all possible GW nodes, set mobility to them and install to Internet
      NodeContainer gwNodes;
      gwNodes.Create (m_satConf->GetGwCount ());
      SetGwMobility (gwNodes);
      internet.Install (gwNodes);

      for ( BeamUserInfoMap_t::iterator info = beamInfos.begin (); info != beamInfos.end (); info++)
        {
          // create UTs of the beam, set mobility to them and install to Internet
          NodeContainer uts;
          uts.Create (info->second.GetUtCount ());
          SetUtMobility (uts, info->first);
          internet.Install (uts);

          for ( uint32_t i = 0; i < info->second.GetUtCount (); i++ )
            {
              // create and install needed users
              m_userHelper->InstallUt (uts.Get (i), info->second.GetUtUserCount (i));
            }

          std::vector<uint32_t> rtnConf = m_satConf->GetBeamConfiguration (info->first, SatEnums::LD_RETURN);
          std::vector<uint32_t> fwdConf = m_satConf->GetBeamConfiguration (info->first, SatEnums::LD_FORWARD);

          /**
           * GW and beam ids are assumed to be the same for both directions
           * currently!
           */
          NS_ASSERT (rtnConf[SatConf::GW_ID_INDEX] == fwdConf[SatConf::GW_ID_INDEX]);
          NS_ASSERT (rtnConf[SatConf::BEAM_ID_INDEX] == fwdConf[SatConf::BEAM_ID_INDEX]);

          // gw index starts from 1 and we have stored them starting from 0
          Ptr<Node> gwNode = gwNodes.Get (rtnConf[SatConf::GW_ID_INDEX] - 1);
          m_beamHelper->Install (uts,
                                 gwNode,
                                 rtnConf[SatConf::GW_ID_INDEX],
                                 rtnConf[SatConf::BEAM_ID_INDEX],
                                 rtnConf[SatConf::U_FREQ_ID_INDEX],
                                 rtnConf[SatConf::F_FREQ_ID_INDEX],
                                 fwdConf[SatConf::U_FREQ_ID_INDEX],
                                 fwdConf[SatConf::F_FREQ_ID_INDEX]);
        }

      m_userHelper->InstallGw (m_beamHelper->GetGwNodes (), gwUsers);

      if (m_packetTraces)
        {
          EnablePacketTrace ();
        }

      m_scenarioCreated = true;
    }

  m_beamHelper->Init ();
}
```


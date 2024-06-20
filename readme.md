```csharp
// A Hive-like System

// each sector has a unique hue
// bees store a path of sectors
// sector[0] is their hive

[CreateAssetMenu]
public class World : ScriptableObject
{
  public List<Light> lights = new List<Light>();
  public List<Sector> sectors = new List<Sector>();
  public List<Bee> bees = new List<Bee>();

  public List<Route> routes = new List<Route>();

  public int maxCapacity = 6;
  public int maxPathCount = 4;
  public float teamCutoff = 0.1666f;

  public float lightStrength = 700;
  public float orbitSpeedMin = 0.5f;
  public float orbitSpeedMax = 1.25f;
  public float orbitRadiusMin = 100;
  public float orbitRadiusMax = 200;

  public delegate void StepAction();
  public static event StepAction OnStep;

  public void NewWorld()
  {
    sectors.Clear();
    bees.Clear();
    lights.Clear();
    sectors.Clear();

    for (int i = 0; i < 100; i++) // SECTORS
    {
      float capacity = Random.value;
      Sector sector = new Sector(
        "sector" + i,
        (float)i / 100,
        new Vector3(Rando(600), Rando(300), Rando(600)),
        (int)(capacity * capacity * maxCapacity)
      );
      sectors.Add(sector);
    }

    for (int i = 0; i < 3; i++) // LIGHTS
    {
      Light light = new Light(
        Vector3.zero,
        Random.rotation,
        Random.Range(0, 360),
        Random.Range(orbitSpeedMin, orbitSpeedMax),
        Random.Range(orbitRadiusMin, orbitRadiusMax)
      ); lights.Add(light);
    }
  }

  public void SimStep(int count)
  {
    for (int i = 0; i < count; i++)
    {
      Step();
    }
  }

  public void Step()
  {
    routes.Clear();

    for (int i = 0; i < lights.Count; i++) // STARS
    {
      Light light = lights[i];
      light.Orbit();
    }

    for (int i = 0; i < sectors.Count; i++) // SECTORS
    {
      Sector sector = sectors[i];

      float dist = 0;
      for (int j = 0; j < lights.Count; j++)
      {
        Light light = lights[j];
        dist += Mathf.Clamp01(Vector3.Distance(sector.position, light.position) / lightStrength);
      }

      float burn = 1 - (dist / 3);
      float freeze = dist / 3;
      float green = 1 - Mathf.Abs(burn - freeze);
      sector.capacity = (int)(green * green * green * maxCapacity);
    }

    if (bees.Count == 0) // SEED
    {
      bees.Add(new Bee(new List<Sector>() { sectors[Random.Range(0, sectors.Count)] }));
    }

    bees.Shuffle(new System.Random());
    for (int i = bees.Count - 1; i >= 0; i--) // BEES
    {
      Bee bee = bees[i];

      switch (bee.state)
      {
        case BeeState.search:
          List<Sector> prospects = sectors.FindAll(
            x => !bee.path.Contains(x)
            && x.capacity > 0
          );

          IEnumerable<Sector> byDist = prospects.OrderBy(x => Vector3.Distance(x.position, bee.Prospect().position));
          prospects = byDist.ToList();

          if (prospects.Count > 0 && bee.path.Count < maxPathCount)
          {
            Sector prospect = prospects[Random.Range(0, Mathf.Min(prospects.Count, 3))];
            bee.path.Add(prospect);

            int taken = bees.FindAll(
              x => x.Prospect() == prospect
              && Team(x.Hive().hue, bee.Hive().hue)
              && x.path.Count > 1
            ).Count;

            if (taken < prospect.capacity)
            {
              Bee beepeer = bees.Find(
                x => Team(x.Hive().hue, bee.Hive().hue)
                && x.Prospect() == prospect
                && x.path.Count > 1
              );

              if (beepeer != null)
                bee.path = new List<Sector>(beepeer.path);

              bee.state = BeeState.forward;
            }
          }
          else
          {
            bee.state = BeeState.forward;
          }
          break;
        case BeeState.forward:
          if (bee.pos < bee.path.Count - 1)
            Warp(bee, 1);

          if (bee.pos == bee.path.Count - 1)
          {
            int taken = bees.FindAll(
              x => x.Prospect() == bee.Prospect()
              && x.path.Count > 1
            ).Count;
            if (taken <= bee.Prospect().capacity)
            {
              bee.state = BeeState.back;
            }
            else
            {
              bees.RemoveAt(i);
              break;
            }

            if (Team(bee.Prospect().hue, bee.Hive().hue))
            {
              bees.Add(new Bee(new List<Sector>() { bee.Prospect() }));
            }

            List<Bee> enemies = bees.FindAll(
              x => !Team(x.Hive().hue, bee.Hive().hue)
              && x.Sector() == bee.Sector()
            );
            if (enemies.Count > 0 && Random.value > 0.5f)
            {
              bees.RemoveAt(i);
              break;
            }
          }
          break;
        case BeeState.back:
          if (bee.pos > 0)
            Warp(bee, -1);

          if (bee.pos == 0)
          {
            bee.state = BeeState.forward;
            bees.Add(new Bee(new List<Sector>() { bee.Hive() }));
          }
          break;
      }
    }

    if (OnStep != null)
    {
      OnStep();
    }
  }

  public void Warp(Bee bee, int direction = 1)
  {
    Sector oldSector = bee.Sector();
    bee.pos += direction;
    routes.Add(new Route(oldSector, bee.Sector(), bee.Hive()));
  }

  public bool Team(float one, float two)
  {
    List<float> values = new List<float>();
    values.Add(Mathf.Abs(one - two));
    float oneUp = Mathf.Abs(1 - one);
    float twoUp = Mathf.Abs(1 - two);
    values.Add(Mathf.Abs(oneUp - twoUp));

    return values.Min() < teamCutoff;
  }

  public float Rando(float value = 1) // NEEDS A NEW HOME...
  {
    return Random.Range(-1f, 1f) * value;
  }
}

[Serializable]
public class Sector
{
  public string name;
  public float hue;
  public Vector3 position;
  public int capacity;

  public Sector(string name, float hue, Vector3 position, int capacity)
  {
    this.name = name;
    this.hue = hue;
    this.position = position;
    this.capacity = capacity;
  }
}

[Serializable]
public class Bee
{
  public List<Sector> path;
  public int pos = 0;
  public BeeState state;

  public Bee(List<Sector> path)
  {
    this.path = path;
    this.pos = 0;

    if (path.Count > 1) { this.state = BeeState.forward; }
    else { this.state = BeeState.search; }
  }

  public Sector Hive() { return path[0]; }
  public Sector Sector() { return path[pos]; }
  public Sector Prospect() { return path[path.Count - 1]; }
}

public enum BeeState { forward, back, search };

[Serializable]
public class Light
{
  public Vector3 barycenter;
  public Quaternion rot;
  public float angle;
  public float speed;
  public float radius;

  public Vector3 position;

  public Light(Vector3 barycenter, Quaternion rot, float angle, float speed, float radius)
  {
    this.barycenter = barycenter;
    this.rot = rot;
    this.angle = angle;
    this.speed = speed;
    this.radius = radius;
    Orbit();
  }

  public Vector3 Orbit()
  {
    float scale = 1 - ((float)radius / 250);
    angle += speed * scale;
    Vector3 pos = rot * Quaternion.AngleAxis(angle, Vector3.up) * Vector3.forward * radius;
    return position = barycenter + pos;
  }
}

[Serializable]
public class Route
{
  public Sector hive;
  public Sector from;
  public Sector to;

  public Route(Sector from, Sector to, Sector hive)
  {
    this.from = from;
    this.to = to;
    this.hive = hive;
  }
}

// I feel just a little bad when I have to clear it, being that it's now alive(in a way)

project.name = null
project.platform[1] = { Oculus VR }
project.release-date = null
```
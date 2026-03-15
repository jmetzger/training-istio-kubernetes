# Features only in VirtualService 

  * Some features are only implemented in VirtualService (like FaultInjection, Retries)

## Vorgehensweise 

   * httpRoute wird für NorthSouth Traffic verwendet (das was ins Cluster reinkommt)
   * Im Mesh kann ich entweder httpRoute oder virtualService verwenden
   * Wenn eine Feature nur in virtualService und noch nicht für die Gateway Api implementiert ist, muss ich virtualService im Mesh verwenden (Beispiel Fault Injection, Retries)

## Warum kann ich im Mesh httpRoute und und VirtualService nicht mischen 

   * Um nicht alles neu zu entwickeln, hat sich istio httpRoute (Mesh) -> in -> virtualService umzuschreiben
   * D.h. für einen Service entweder virtualService oder httpRoute
   * Wann immer möglich (weil alle Feature, die ich brauche drin sind), würde ich httpRoute nehmen 

Per aggiungere un nuovo dominio (es. ANC) basta: 
    aggiungere il remote in vite.config.ts e una riga nel mfeRegistry. Il routing e la voce di menu si generano automaticamente dall'array.
    Dove prenderlo — Module Federation (build/config)
        In case-platform-frontend\apps\shell\vite.config.ts della Shell è dichiarato il remote:
        remotes: {
        pgcc_mfe: {
            entry: env.PGCC_MFE_ENTRY ?? `http://localhost:${env.PGCC_MFE_PORT}/remoteEntry.js`,
        },
        }
        In sviluppo PGCC_MFE_PORT=3001 (dal .env), quindi pgcc_mfe → http://localhost:3001/remoteEntry.js. Quel file è il "manifesto" che il MFE PGCC pubblica con i componenti che espone (PgccHome, PgccMenu). In produzione userebbe PGCC_MFE_ENTRY instradato da APISIX
Cosa montare e su che rotta — il registry (app)
    In case-platform-frontend\apps\shell\src\mfe-registry.ts c'è la lista dei domini:

        export const mfeRegistry: MfeEntry[] = [
        {
            path: '/pgcc',
            label: 'PGCC',
            Page: lazy(() => import('pgcc_mfe/PgccHome')),
            Menu: lazy(() => import('pgcc_mfe/PgccMenu')),
        },
        ]
        import('pgcc_mfe/PgccHome') non è un import locale: Module Federation lo risolve a runtime scaricando il remoteEntry.js del remote pgcc_mfe definito al punto 1.
Come diventa navigazione — App.tsx
    case-platform-frontend\apps\shell\src\App.tsx itera il registry per generare rotte e redirect di default:
        const defaultPath = mfeRegistry[0]?.path ?? '/'   // → '/pgcc'
        <Route path='/' element={<Navigate to={defaultPath} replace />} />
        {mfeRegistry.map(({ path, Page }) => <Route path={path} element={<Page/>} />)}
    Quindi entrando su http://localhost:3000/ → redirect automatico a /pgcc (primo elemento del registry) → carica PgccHome dal MFE.